# ADR-002: Integridade e auditabilidade das evidências como característica arquitetural prioritária

- **Status:** Aceito
- **Data:** 13/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Rascunho elaborado com apoio de IA em sala, sujeito à revisão e aprovação da equipe
- **Revisão 1 (17/10/2026):** Assinatura digital do manifesto, hash do manifesto no e-mail de conclusão, metadados técnicos da captura, navegação remota no vídeo, novos estados do registro e metas de verificação. Revisão feita com apoio de IA fora de sala, com autorização excepcional do professor.

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
2. **Navegação remota no vídeo de navegação** (REQ08). O usuário navega no navegador que roda no worker: a tela é transmitida para o navegador dele e seus cliques e teclas são enviados ao worker (transmissão de tela do Chromium por WebSocket). O que é gravado sai do worker; o dispositivo do usuário só exibe a tela e envia comandos. As ações do usuário são registradas na trilha de auditoria.
3. **Hash SHA-256 de cada arquivo** (imagem e vídeo), calculado pelo worker logo após a captura, antes de qualquer armazenamento.
4. **Manifesto do registro**: documento com id do registro, usuário, URL, data/hora de início e fim, lista de arquivos com nome, tamanho e hash. O hash do próprio manifesto também é registrado.
5. **Metadados técnicos da captura no manifesto:** endereço IP do servidor do site, resolução de DNS, certificado TLS do site, cabeçalhos HTTP da resposta, código HTML da página capturada e versão do navegador. Esses dados ajudam a demonstrar que o conteúdo veio de fato daquele site.
6. **Assinatura digital do manifesto** com a chave privada do Veridit, guardada num cofre fora dos workers e do banco. Sem a chave, nem quem tem acesso ao banco e ao storage consegue trocar arquivos e hashes juntos e gerar um manifesto válido. Qualquer pessoa pode conferir a assinatura com a chave pública publicada pelo Veridit.
7. **Hash do manifesto no e-mail de conclusão** (REQ10, REQ11). O e-mail funciona como um registro externo e datado, guardado pelo usuário e fora do controle do Veridit.
8. **Data e hora vindas do servidor**, sincronizadas por NTP e gravadas em UTC. O relógio do cliente nunca é usado. O carimbo do tempo ICP-Brasil fica previsto como evolução.
9. **Armazenamento imutável** dos arquivos em object storage com versionamento e bloqueio de escrita. Um arquivo de evidência gravado não pode ser sobrescrito nem editado.
10. **Relatório (REQ14) e zip (REQ15) com o manifesto, a assinatura e os hashes**, para que qualquer pessoa possa recalcular os hashes, conferir a assinatura e verificar se os arquivos estão íntegros.
11. **Trilha de auditoria somente de inclusão**: início, conclusão, falha, geração de relatório e download, cada evento com usuário, data/hora e identificador de rastreio.
12. **Máquina de estados do registro:** `NA_FILA`, `EM_ANDAMENTO`, `CONCLUIDO`, `FALHOU` e `CANCELADO` (o usuário desistiu). Só registros `CONCLUIDO` geram relatório e zip, o que evita prova incompleta.

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
| Verificação independente da integridade (qualquer pessoa recalcula o hash) | **Armazenamento:** arquivos imutáveis e versionados não podem ser compactados nem substituídos, e o volume só cresce |
| Rastreabilidade completa de quem fez o quê e quando | **Desempenho:** calcular hash de vídeos grandes e gerar o manifesto aumenta o tempo até concluir o registro |
| Prevenção de prova incompleta (máquina de estados) | **Conflito com a LGPD:** a imutabilidade se choca com o direito de exclusão do titular, então é preciso definir política de retenção e expurgo controlado (ver ADR-001) |
| | **Usabilidade:** o usuário não pode editar, cortar ou corrigir uma captura e precisa refazer o registro, consumindo novos créditos |
| | **Complexidade:** sincronização de relógio, trilha de auditoria e verificação de hashes adicionam componentes e testes |

## Consequências

**Positivas**
- Relatórios e zips podem ser verificados por terceiros, como peritos, juízes e a parte contrária.
- Fica pronto o caminho para incluir carimbo do tempo ICP-Brasil sem mudar a estrutura.
- A trilha de auditoria também serve à segurança (ADR-001) e ao diagnóstico de falhas.

**Negativas / riscos**
- O custo de armazenamento cresce sempre e precisa de política de retenção definida com o produto.
- Se o relógio dos workers não estiver sincronizado, a validade de todas as evidências fica em risco, então a sincronização precisa ser monitorada.
- Qualquer serviço que precise alterar evidências exigirá uma nova ADR que substitua esta.
- A chave privada de assinatura vira um ativo crítico: se vazar, manifestos podem ser forjados. Ela precisa ficar em cofre, com acesso restrito e plano de troca.
- A transmissão de tela no vídeo de navegação consome banda e exige baixa latência entre worker e usuário, o que influencia o dimensionamento (ADR-004).
