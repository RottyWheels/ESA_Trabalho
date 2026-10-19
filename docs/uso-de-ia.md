# Uso de IA no projeto Veridit

Registro de onde e como a equipe usou inteligência artificial no projeto arquitetural do Veridit, para dar transparência ao professor e aos avaliadores. A IA foi usada como apoio de análise e revisão em seis etapas. **Todas as decisões finais foram tomadas pelos membros da equipe.**

Ferramentas usadas:

- **Claude (assistente de IA):** análise das características, comparação de estilos arquiteturais, revisão das [ADRs](adr/), revisão dos [componentes candidatos](componentes.md), proposta do [mapeamento de tecnologias](tecnologias.md), conferência das [táticas](taticas.md) e esboço dos diagramas.
- **Graphify:** geração de um grafo de conhecimento a partir das ADRs, do [README](../README.md), dos componentes, das tecnologias e dos PDFs de requisitos e do enunciado, usado para consultas mais baratas e para checar a coerência entre os documentos.

## 1. Análise das características arquiteturais

A IA foi usada para revisar as características que a equipe escolheu e apontar falhas ou incoerências. A lista inicial e a seleção do top 4 foram feitas pela equipe.

- **Como:** a IA recebeu o documento de requisitos e a lista de características e verificou se cada uma tinha base nos requisitos, se havia sobreposição entre elas e se faltava alguma.
- **Exemplos do que foi apontado:** a separação entre segurança (quem acessa) e integridade das evidências (provar que a prova não mudou); a distinção entre confiabilidade transacional (estar correto) e disponibilidade (estar no ar); e as tensões entre características, como imutabilidade × direito de exclusão da LGPD.
- **Papel da equipe:** avaliou cada apontamento, decidiu quais aceitar e definiu a ordem final do top 10 e do top 4.

## 2. Escolha do estilo arquitetural

A IA foi usada para comparar os estilos arquiteturais candidatos e levantar os principais trade-offs de cada um. A equipe escolheu o estilo service-based, registrado na [ADR-011](adr/ADR-011-estilo-arquitetural-service-based.md).

- **Estilos comparados:** monólito em camadas, monólito modular, service-based, microsserviços, orientada a eventos e serverless.
- **Como:** a IA avaliou cada estilo frente às características prioritárias e à restrição de viabilidade (equipe pequena, prazo curto). A tabela comparativa da ADR-011 é uma avaliação da equipe, inspirada na classificação de Richards e Ford.
- **Papel da equipe:** decidiu o estilo final e manteve o monólito modular como plano B, levando em conta a experiência dos membros e o prazo de implementação.

## 3. Análise, correção e aprimoramento das ADRs

As 11 ADRs passaram por 44 revisões com apoio de IA depois da primeira versão, todas registradas no cabeçalho de cada ADR e em commits `docs(adr): ...`.

**Como a IA foi usada**

1. **Completar cada ADR:** sugerir mecanismos, alternativas e metas de verificação que faltavam.
2. **Checar coerência entre ADRs:** encontrar contradições, como nomes de eventos diferentes para a mesma coisa ou a oferta de Pix quando o Mercado Pago cai, sendo que o Pix depende dele.
3. **Aplicar um checklist de revisão:** itens como captura interrompida, eventos de consumo e devolução de créditos e padronização do cabeçalho.
4. **Alinhar com os demais artefatos:** ajustar as ADRs à lista de [componentes candidatos](componentes.md).

**Como cada revisão é registrada**

Cada ADR tem no cabeçalho o campo **Apoio de IA**, que diz se a versão inicial foi feita em sala ou fora de sala com autorização excepcional do professor. Abaixo dele, cada **Revisão N** traz a data, o que mudou e se teve apoio de IA.

| ADR | Versão inicial | Revisões |
|---|---|---|
| [ADR-001](adr/ADR-001-seguranca.md) Segurança | Em sala | 5 |
| [ADR-002](adr/ADR-002-integridade-auditabilidade-evidencias.md) Integridade das evidências | Em sala | 5 |
| [ADR-003](adr/ADR-003-confiabilidade-transacional-pagamentos.md) Confiabilidade transacional | Em sala | 4 |
| [ADR-004](adr/ADR-004-escalabilidade-elasticidade-captura.md) Escalabilidade da captura | Em sala | 4 |
| [ADR-005](adr/ADR-005-disponibilidade-tolerancia-falhas.md) Disponibilidade | Fora de sala, com autorização | 4 |
| [ADR-006](adr/ADR-006-durabilidade-recuperacao-evidencias.md) Durabilidade | Fora de sala, com autorização | 3 |
| [ADR-007](adr/ADR-007-conformidade-lgpd.md) LGPD | Fora de sala, com autorização | 4 |
| [ADR-008](adr/ADR-008-interoperabilidade.md) Interoperabilidade | Fora de sala, com autorização | 5 |
| [ADR-009](adr/ADR-009-extensibilidade-manutenibilidade.md) Extensibilidade | Fora de sala, com autorização | 5 |
| [ADR-010](adr/ADR-010-desempenho.md) Desempenho | Fora de sala, com autorização | 2 |
| [ADR-011](adr/ADR-011-estilo-arquitetural-service-based.md) Estilo service-based | Fora de sala, com autorização | 3 |

Toda sugestão da IA foi lida e aprovada pela equipe antes de entrar na ADR.

## 4. Componentes, tecnologias e táticas

A IA também apoiou os artefatos derivados das ADRs.

- **[Componentes candidatos](componentes.md):** a IA cruzou os 21 componentes com as ADRs e apontou lacunas, como eventos sem componente responsável e componentes sem a ADR que os justifica. As correções foram registradas em revisões das ADRs e da lista de componentes.
- **[Tecnologias](tecnologias.md):** a IA propôs o mapeamento das tecnologias gratuitas por camada e depois verificou, pelo grafo, se todo componente tinha tecnologia definida. Isso revelou lacunas como o ambiente isolado de captura (C20) sem ferramenta. A linguagem e os frameworks continuam como proposta, a ser registrada em ADR em sala, assim como o corte do MinIO de nó único no MVP (ADR-006).
- **[Táticas](taticas.md):** extraídas das ADRs pela equipe; a IA conferiu se cada tática e o número do item correspondiam à seção "Decisão" da ADR.
- **Papel da equipe:** decidiu a granularidade dos componentes, os componentes críticos e a pilha de tecnologias.

## 5. Graphify: custo de tokens e coerência entre ADRs

O Graphify transformou o repositório num grafo de conhecimento, gerado em 17/10/2026 e atualizado em 18/10/2026. A versão atual tem 311 nós, 1.170 relações e 10 comunidades temáticas. A pasta `graphify-out/` fica só na máquina local e não é versionada. Ele serviu a dois propósitos.

**Reduzir o custo de tokens**

- O grafo foi gerado uma vez e depois atualizado de forma incremental: só os arquivos alterados são relidos.
- As perguntas à IA usam consultas ao grafo (`graphify query`, `path` e `explain`), que devolvem só o trecho relevante, em vez de reler todos os documentos.
- A regra fica num arquivo de instruções local do assistente (`CLAUDE.md`), fora do repositório: consultar o grafo antes de ler arquivos e atualizá-lo após mudanças.
- A construção e as atualizações do grafo consumiram cerca de 1 milhão de tokens no total, em 8 execuções.

**Analisar coerência e pontos cegos**

- **Nós centrais:** a ADR-001 (58 conexões), a ADR-011 (48) e a ADR-005 (44) ligam quase todas as comunidades, o que confirma o papel delas como base da proposta. O documento de tecnologias (69) passou a ser a maior ponte entre as comunidades.
- **Nós pouco conectados:** apontou 33 itens com no máximo uma conexão, como o diagrama de módulos, as telas no Figma e as entidades do Apêndice A, indicando entregáveis e conceitos ainda sem ligação com as ADRs.
- **Relações inferidas:** 247 relações (21%) foram deduzidas pelo modelo e marcadas para a equipe confirmar ou descartar.
- **Inconsistências:** ajudou a localizar divergências entre documentos, como o REQ11 duplicado no documento de requisitos.

## 6. Esboços dos diagramas

A IA gerou esboços iniciais dos diagramas do sistema, que a equipe usou como ponto de partida. Os diagramas entregues foram ajustados e finalizados pelos membros.

- **Como:** a partir das ADRs e da divisão em seis serviços, a IA propôs a disposição dos componentes, das conexões entre eles e dos pacotes de cada serviço.
- **Papel da equipe:** conferiu cada esboço com as ADRs, corrigiu nomes e relações e decidiu o nível de detalhe de cada diagrama.

## Decisões finais da equipe

A IA foi usada como apoio de análise; todas as decisões finais foram tomadas pelos membros da equipe.

- As recomendações da IA foram tratadas como sugestões, não como respostas definitivas.
- Cada decisão levou em conta a experiência e a preferência da equipe, além do prazo e do escopo do trabalho.
- Sugestões que não se encaixavam no projeto foram descartadas ou adaptadas.
- Os membros revisaram e aprovaram todo o conteúdo gerado com apoio de IA antes de incluí-lo no repositório.

## Resumo por etapa

| Etapa | Uso da IA | Decisão da equipe |
|---|---|---|
| Características arquiteturais | Apontar falhas, sobreposições e incoerências | Top 10 e top 4 finais |
| Estilo arquitetural | Comparar estilos e trade-offs | Service-based, com monólito modular como plano B |
| ADRs | Completar, corrigir e checar coerência (44 revisões registradas) | Aprovação de cada revisão |
| Componentes, tecnologias e táticas | Cruzar com as ADRs, propor tecnologias gratuitas e apontar lacunas | Componentes críticos e pilha de tecnologias |
| Graphify | Reduzir tokens e mapear conexões entre os documentos | Confirmar ou descartar relações inferidas |
| Diagramas | Esboços iniciais | Versões finais entregues |
