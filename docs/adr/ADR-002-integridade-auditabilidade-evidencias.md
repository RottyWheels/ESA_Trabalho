# ADR-002: Integridade e auditabilidade das evidências como característica arquitetural prioritária

- **Status:** Aceito
- **Data:** 13/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Elaborado com apoio de IA em sala; revisado e aprovado pela equipe
- **Revisão 1 (17/10/2026):** Assinatura digital do manifesto, hash do manifesto no e-mail de conclusão, metadados técnicos da captura, navegação remota no vídeo, novos estados do registro e metas de verificação. Revisão feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 2 (17/10/2026):** Correções de coerência com as demais ADRs: transmissão do vídeo pelo proxy do Gateway, assinatura pedida pela API de Captura (o worker não acessa a chave), exclusão só pelo expurgo da ADR-007, auditoria da geração do link de download e referências à ADR-007. Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 3 (17/10/2026):** Checklist de revisão (A1: captura interrompida não é refeita; A3: eventos `CapturaExecutada` e `CapturaConcluida`; A5: `NA_FILA` só na captura de tela; D1: cabeçalho). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 4 (17/10/2026):** Coerência com as demais ADRs: só a API de Captura muda o estado do registro; início da captura de tela por pedido e resposta no broker e evento `CapturaComErro` para falhas com o worker vivo (item 12). Feita com apoio de IA fora de sala.
- **Revisão 5 (17/10/2026):** Agrupamento dos componentes candidatos: trilha de auditoria no Serviço de Registros e Relatórios, alimentada pelo evento `EventoAuditoria` (item 11). Feita com apoio de IA fora de sala.

## Contexto

O Veridit existe para que o usuário, em geral um advogado, registre conteúdo da internet (tela ou vídeo de navegação) e o use como **prova**. O valor do produto não está na captura em si, que qualquer ferramenta de print faz, e sim na possibilidade de **demonstrar** que o conteúdo:

- veio da URL informada;
- foi capturado na data e hora registradas;
- não foi alterado depois da captura;
- foi capturado por um usuário identificado.

O Apêndice A pede que o relatório traga: id do relatório e do registro gerados pelo sistema, usuário e CPF, data/hora de início e fim, duração, imagens capturadas, site e URL navegada, nome do arquivo gerado pelo sistema e tamanho do arquivo. Esses campos só têm valor se não puderem ser falsificados.

## Requisitos e restrições relacionados

- **REQ08:** Iniciar registro de conteúdo (endereço do site e título)
- **REQ09 e REQ11:** Concluir gravação de vídeo e captura de tela
- **REQ13:** Visualizar detalhes do registro (id, datas, tipos e quantidade de arquivos, usuário responsável)
- **REQ14:** Gerar relatório do registro (campos do Apêndice A)
- **REQ15:** Gerar arquivo zip com os arquivos capturados
- **Apêndice A:** Campos de "Registros realizados", "Detalhes dos registros" e "Relatório do registro de conteúdo"
- **Restrição de domínio:** As evidências podem ser usadas em processos judiciais, e a parte contrária pode questionar sua autenticidade

## Decisão

Adotar **Integridade e auditabilidade das evidências** como característica arquitetural prioritária (2ª do top 4), implementada pelos seguintes mecanismos:

1. **Captura feita no servidor**, em workers controlados pelo Veridit com navegador sem interface. O conteúdo nunca vem do dispositivo do usuário. Cada captura usa um navegador novo, sem cookies nem dados de capturas anteriores.
2. **Navegação remota no vídeo de navegação** (REQ08). O usuário navega no navegador que roda no worker: a tela é transmitida para o navegador dele e seus cliques e teclas são enviados ao worker (transmissão de tela do Chromium por WebSocket). O que é gravado sai do worker; o dispositivo do usuário só exibe a tela e envia comandos. As ações do usuário são registradas na trilha de auditoria. A transmissão passa por um proxy de sessão no Gateway, autenticado com o token do usuário; o worker não recebe conexões diretas da internet (ADR-001).
3. **Hash SHA-256 de cada arquivo** (imagem e vídeo), calculado pelo worker logo após a captura, antes de qualquer armazenamento.
4. **Manifesto do registro**: documento com id do registro, usuário, URL, data/hora de início e fim, lista de arquivos com nome, tamanho e hash. O hash do próprio manifesto também é registrado.
5. **Metadados técnicos da captura no manifesto:** endereço IP do servidor do site, resolução de DNS, certificado TLS do site, cabeçalhos HTTP da resposta, código HTML da página capturada e versão do navegador. Esses dados ajudam a demonstrar que o conteúdo veio de fato daquele site.
6. **Assinatura digital do manifesto** com a chave privada do Veridit, guardada num cofre fora dos workers e do banco. Quem pede a assinatura é a API de Captura, ao receber o evento `CapturaExecutada` do worker e depois de conferir os hashes do manifesto com os checksums que o object storage registrou na gravação. O worker nunca tem acesso à chave, para que um worker comprometido por um site malicioso não consiga assinar manifestos falsos. Sem a chave, nem quem tem acesso ao banco e ao storage consegue trocar arquivos e hashes juntos e gerar um manifesto válido. Qualquer pessoa pode conferir a assinatura com a chave pública publicada pelo Veridit. Só depois de assinar, a API marca o registro como `CONCLUIDO` e publica o evento `CapturaConcluida` para os demais serviços.
7. **Hash do manifesto no e-mail de conclusão** (REQ10, REQ11). O e-mail funciona como um registro externo e datado, guardado pelo usuário e fora do controle do Veridit.
8. **Data e hora vindas do servidor**, sincronizadas por NTP e gravadas em UTC. O relógio do cliente nunca é usado. O carimbo do tempo ICP-Brasil fica previsto como evolução.
9. **Armazenamento imutável** dos arquivos em object storage com versionamento e bloqueio de escrita. Um arquivo de evidência gravado não pode ser sobrescrito nem editado. A única exclusão permitida é o expurgo controlado da ADR-007, ao fim do prazo de retenção ou a pedido do titular.
10. **Relatório (REQ14) e zip (REQ15) com o manifesto, a assinatura e os hashes**, para que qualquer pessoa possa recalcular os hashes, conferir a assinatura e verificar se os arquivos estão íntegros.
11. **Trilha de auditoria somente de inclusão**: início, conclusão, falha, geração de relatório, geração de link de download e expurgos, cada evento com usuário, data/hora e identificador de rastreio. A trilha fica no Serviço de Registros e Relatórios, que já consome os eventos da Captura e gera relatórios e links de download. Os demais serviços publicam o evento `EventoAuditoria` na fila, e Registros e Relatórios o grava numa tabela que só aceita inclusão: nenhum serviço altera os registros dela, que só saem pelo prazo de retenção (ADR-007).
12. **Máquina de estados do registro:** `NA_FILA`, `EM_ANDAMENTO`, `CONCLUIDO`, `FALHOU` e `CANCELADO` (o usuário desistiu). `NA_FILA` existe só na captura de tela; o vídeo começa direto em `EM_ANDAMENTO`, quando o navegador é alocado (ADR-004). Só a API de Captura muda o estado do registro. Na captura de tela, o worker pede o início pelo broker (pedido e resposta, sem acesso ao banco, ADR-001); a API passa o registro de `NA_FILA` para `EM_ANDAMENTO` numa única operação condicional e só autoriza se ele ainda estiver `NA_FILA`. Se o site não carregar ou a captura der erro, o worker publica `CapturaComErro` e a API marca o registro como `FALHOU`, sem esperar o prazo de expiração (ADR-005). Só registros `CONCLUIDO` geram relatório e zip, o que evita prova incompleta. Uma captura interrompida não é refeita automaticamente: vira `FALHOU`, o crédito volta e o usuário refaz o registro (ADR-003, ADR-005).

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Arquivos de registros concluídos com hash no manifesto | 100% |
| Manifestos com assinatura digital válida | 100% |
| Alteração de um byte em qualquer arquivo ou no manifesto | Detectada pela verificação de hash ou de assinatura (teste automatizado) |
| E-mails de conclusão com o hash do manifesto | 100% |
| Diferença entre o relógio dos workers e a referência NTP | Menos de 1 segundo, com alerta acima disso |

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Captura no navegador do usuário (extensão ou gravação de tela local) | Mais barata e simples, mas o usuário controla o ambiente e pode adulterar o conteúdo, o DOM ou o relógio. A prova perde credibilidade. |
| Guardar só os arquivos, sem hash nem manifesto | Simples, mas não permite provar que os arquivos não mudaram. Tira o valor jurídico do produto. |
| Carimbo do tempo ICP-Brasil já na primeira versão | Dá maior força jurídica, mas tem custo por carimbo, exige contratar uma Autoridade de Carimbo do Tempo e aumenta a complexidade. Fica como evolução. Na primeira versão, a assinatura do manifesto e o hash enviado por e-mail dão uma garantia intermediária. |
| Integridade baseada só em hashes guardados pelo próprio Veridit | Mais simples, mas quem tem acesso interno pode trocar arquivo e hash juntos, e a parte contrária pode alegar exatamente isso. Por isso a assinatura e o registro externo por e-mail. |
| Registro dos hashes em blockchain pública | Garante imutabilidade externa, mas tem custo por transação, dependência externa e complexidade desproporcional ao escopo atual. |
| Integridade fora do top 4, tratada como detalhe de implementação | É o diferencial do produto. Se não for característica arquitetural, decisões de armazenamento e captura podem comprometê-la sem que ninguém perceba. |

## Justificativa

Integridade e auditabilidade entram no top 4 porque são **o motivo de o sistema existir**. Um registro que pode ser contestado vale tanto quanto um print comum. Diferente de características genéricas como desempenho ou manutenibilidade, esta vem diretamente do domínio jurídico e dos campos do Apêndice A.

A arquitetura service-based ajuda nessa decisão. O Serviço de Captura é o único que produz evidências e calcula hashes. O Serviço de Registros e Relatórios só lê e empacota o que foi gravado, sem poder alterá-lo. Essa divisão facilita aplicar e auditar a regra de imutabilidade.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Valor probatório e credibilidade jurídica do produto | **Custo de infraestrutura:** a captura no servidor exige navegadores rodando nos workers, em vez de usar o dispositivo do usuário |
| Verificação independente da integridade (qualquer pessoa recalcula o hash) | **Armazenamento:** arquivos imutáveis e versionados não podem ser compactados nem substituídos, e o volume cresce até o fim do prazo de retenção (ADR-007) |
| Rastreabilidade completa de quem fez o quê e quando | **Desempenho:** calcular hash de vídeos grandes e gerar o manifesto aumenta o tempo até concluir o registro |
| Prevenção de prova incompleta (máquina de estados) | **Conflito com a LGPD:** a imutabilidade se choca com o direito de exclusão do titular, resolvido pela retenção com prazo e pelo expurgo controlado (ADR-007) |
| | **Usabilidade:** o usuário não pode editar, cortar ou corrigir uma captura e precisa refazer o registro, consumindo novos créditos |
| | **Complexidade:** sincronização de relógio, trilha de auditoria e verificação de hashes adicionam componentes e testes |

## Consequências

**Positivas**
- Relatórios e zips podem ser verificados por terceiros, como peritos, juízes e a parte contrária.
- Fica pronto o caminho para incluir carimbo do tempo ICP-Brasil sem mudar a estrutura.
- A trilha de auditoria também serve à segurança (ADR-001) e ao diagnóstico de falhas.

**Negativas / riscos**
- O custo de armazenamento cresce até o fim do prazo de retenção (ADR-007), que ainda precisa ser validado com o produto.
- Se o relógio dos workers não estiver sincronizado, a validade de todas as evidências fica em risco, então a sincronização precisa ser monitorada.
- Qualquer serviço que precise alterar evidências exigirá uma nova ADR que substitua esta.
- A chave privada de assinatura vira um ativo crítico: se vazar, manifestos podem ser forjados. Ela precisa ficar em cofre, com acesso restrito e plano de troca.
- A transmissão de tela no vídeo de navegação consome banda e exige baixa latência entre worker e usuário, o que influencia o dimensionamento (ADR-004).
