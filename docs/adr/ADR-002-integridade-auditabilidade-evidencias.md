# ADR-002: Integridade e auditabilidade das evidências como característica arquitetural prioritária

- **Status:** Proposto
- **Data:** 13/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Rascunho elaborado com apoio de IA em sala, sujeito à revisão e aprovação da equipe

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

1. **Captura feita no servidor**, em workers controlados pelo Veridit com navegador sem interface. O conteúdo nunca vem do dispositivo do usuário.
2. **Hash SHA-256 de cada arquivo** (imagem e vídeo), calculado pelo worker logo após a captura, antes de qualquer armazenamento.
3. **Manifesto do registro**: documento com id do registro, usuário, URL, data/hora de início e fim, lista de arquivos com nome, tamanho e hash. O hash do próprio manifesto também é registrado.
4. **Data e hora vindas do servidor**, sincronizadas por NTP e gravadas em UTC. O relógio do cliente nunca é usado. O carimbo do tempo ICP-Brasil fica previsto como evolução.
5. **Armazenamento imutável** dos arquivos em object storage com versionamento e bloqueio de escrita. Um arquivo de evidência gravado não pode ser sobrescrito nem editado.
6. **Relatório (REQ14) e zip (REQ15) com o manifesto e os hashes**, para que qualquer pessoa possa recalcular os hashes e verificar se os arquivos estão íntegros.
7. **Trilha de auditoria somente de inclusão**: início, conclusão, falha, geração de relatório e download, cada evento com usuário, data/hora e identificador de rastreio.
8. **Máquina de estados do registro** (por exemplo `EM_ANDAMENTO`, `CONCLUIDO`, `FALHOU`). Só registros `CONCLUIDO` geram relatório e zip, o que evita prova incompleta.

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Captura no navegador do usuário (extensão ou gravação de tela local) | Mais barata e simples, mas o usuário controla o ambiente e pode adulterar o conteúdo, o DOM ou o relógio. A prova perde credibilidade. |
| Guardar só os arquivos, sem hash nem manifesto | Simples, mas não permite provar que os arquivos não mudaram. Tira o valor jurídico do produto. |
| Carimbo do tempo ICP-Brasil já na primeira versão | Dá maior força jurídica, mas tem custo por carimbo, exige contratar uma Autoridade de Carimbo do Tempo e aumenta a complexidade. Fica como evolução. |
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
