# ADR-011: Estilo arquitetural service-based

- **Status:** Aceito
- **Data:** 17/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Rascunho elaborado com apoio de IA fora de sala, com autorização excepcional do professor, sujeito à revisão e aprovação da equipe

## Contexto

O Veridit precisa de um estilo arquitetural que atenda às quatro características prioritárias (ADR-001 a ADR-004) e às demais (ADR-005 a ADR-010), dentro de uma restrição forte de viabilidade: equipe acadêmica pequena, prazo curto e orçamento limitado de infraestrutura.

O sistema tem partes com necessidades muito diferentes:

- **Captura** (REQ08 a REQ11): pesada, com navegadores no servidor e carga irregular. Precisa escalar sozinha (ADR-004).
- **Créditos e pagamentos** (REQ05 a REQ07): envolve dinheiro e exige transações ACID e isolamento dos dados de faturamento (ADR-001, ADR-003).
- **Identidade, consulta, relatórios e e-mails** (REQ01 a REQ04, REQ12 a REQ15): leves e de carga estável, mas a consulta às provas precisa continuar disponível mesmo quando a captura falha (ADR-005).

Vários fluxos já são assíncronos por natureza: confirmação de pagamento por webhook, captura demorada, geração de relatório e zip, e e-mails (REQ07, REQ10, REQ11).

## Requisitos e restrições relacionados

- **REQ01 a REQ15:** Todos os requisitos funcionais, distribuídos entre os serviços
- **Atributo de qualidade:** Segurança (acesso só por usuários cadastrados)
- **D1 e D2:** Evolução prevista em tipos de usuário e pacotes de créditos
- **ADR-001 a ADR-004:** Características prioritárias
- **ADR-005 a ADR-010:** Demais características arquiteturais
- **Restrição do projeto:** Viabilidade (equipe pequena, prazo curto, ferramentas open source executáveis localmente)

## Decisão

Adotar o estilo **service-based** (arquitetura baseada em serviços), com processamento assíncrono por fila nas operações demoradas.

**Serviços de domínio**, de granularidade grossa e implantados de forma independente:

| Serviço | Responsabilidade | Requisitos |
|---|---|---|
| Interface Web + API Gateway | Interface do usuário, roteamento e validação do token | Todos |
| Identidade | Cadastro, login, logout, recuperação de senha e perfis | REQ01 a REQ04, D1 |
| Créditos e Pagamentos | Pacotes, faturamento, pagamento, webhooks e livro-razão de créditos | REQ05, REQ06, D2 |
| Captura | API de captura e workers com navegador no servidor | REQ08, REQ09, REQ11 |
| Registros e Relatórios | Listagem, detalhes, relatório e zip | REQ12 a REQ15 |
| Notificação | Envio de e-mails a partir de eventos | REQ07, REQ10, REQ11 |

**Topologia:**

1. **Um único banco PostgreSQL**, com um schema por serviço. Cada serviço acessa só o próprio schema, com usuário de banco próprio (ADR-009). O schema de pagamentos tem credenciais isoladas (ADR-001).
2. **Comunicação síncrona** por HTTP com JSON entre a interface, o Gateway e os serviços, para consultas e comandos rápidos.
3. **Comunicação assíncrona** por fila (RabbitMQ) para captura, geração de relatório e zip, confirmação de pagamento e e-mails (ADR-003, ADR-004, ADR-005).
4. **Object storage** (MinIO, compatível com S3) para os arquivos de evidência, relatórios e zips (ADR-002, ADR-006).
5. **Escala por serviço:** só os workers de Captura e de Registros e Relatórios escalam horizontalmente. Os demais serviços rodam com uma instância, exceto Gateway e Identidade, com duas (ADR-005).
6. **Execução com Docker Compose** na primeira versão, com todo o sistema executável localmente.

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Implantação independente | Qualquer serviço pode ser atualizado sem reiniciar os outros |
| Isolamento de falhas | Parar todos os workers de captura não afeta login, listagem e download |
| Escala seletiva | Aumentar os workers de captura sem replicar nenhum outro serviço |
| Acoplamento pelo banco | Nenhum serviço acessa o schema de outro (bloqueado por permissões) |
| Viabilidade | O sistema completo sobe localmente com um único comando do Docker Compose |

## Alternativas consideradas

A comparação abaixo é uma avaliação da equipe para o caso do Veridit, inspirada na classificação de estilos arquiteturais de Richards e Ford (*Fundamentos da Arquitetura de Software*).

| Característica | Monólito em camadas | Monólito modular | **Service-based** | Microsserviços | Orientada a eventos |
|---|---|---|---|---|---|
| Segurança (isolamento dos pagamentos) | Baixa | Baixa | **Alta** | Alta | Média |
| Integridade das evidências (responsabilidade única pela captura) | Média | Média | **Alta** | Alta | Média |
| Confiabilidade transacional (ACID local) | Alta | Alta | **Alta** | Baixa | Baixa |
| Escalabilidade da captura | Baixa | Baixa | **Alta** | Alta | Alta |
| Disponibilidade e isolamento de falhas | Baixa | Baixa | **Alta** | Alta | Alta |
| Manutenibilidade e extensibilidade | Baixa | Média | **Alta** | Alta | Média |
| Simplicidade | Alta | Alta | **Média** | Baixa | Baixa |
| Viabilidade (custo e prazo) | Alta | Alta | **Alta** | Baixa | Baixa |

| Alternativa | Motivo da rejeição |
|---|---|
| Monólito em camadas | Mais simples e barato, mas escala como um bloco só: para atender um pico de captura, replica-se também login, pagamento e consulta. Uma falha ou um deploy com erro na captura derruba o sistema inteiro, inclusive o acesso às provas. |
| Monólito modular | Melhora a organização do código, mas continua com um único processo e uma única implantação. Mantém os problemas de escala e de isolamento de falhas da captura. Fica como **plano B** se o prazo de implementação apertar, já que os mesmos módulos podem ser separados em serviços depois. |
| Microsserviços | Máxima independência e escalabilidade, mas exige um banco por serviço, o que obriga a usar consistência eventual e sagas no fluxo de créditos (contrário à ADR-003), além de orquestração, rastreamento distribuído e versionamento de muitos contratos. O custo operacional é desproporcional a 15 requisitos e a uma equipe pequena. |
| Arquitetura orientada a eventos pura | Ótima escalabilidade e desacoplamento, mas torna os fluxos difíceis de acompanhar e testar, e também exige consistência eventual no saldo de créditos. Os fluxos assíncronos do Veridit são poucos e bem definidos; uma fila entre serviços atende sem tornar tudo orientado a eventos. |
| Serverless (funções) | Escala automática e paga pelo uso, mas o limite de tempo de execução e o *cold start* do navegador inviabilizam a captura de vídeo (ADR-004) e aumentam a dependência de um provedor de nuvem. |

## Justificativa

O service-based é o estilo que atende às quatro características prioritárias **sem violar a restrição de viabilidade**:

- **Segurança (ADR-001):** pagamentos e dados de faturamento ficam num serviço com schema e credenciais isolados, e o Gateway é a única porta de entrada.
- **Integridade (ADR-002):** só o Serviço de Captura produz evidências e calcula hashes; Registros e Relatórios apenas lê e empacota. A divisão facilita aplicar e auditar a regra de imutabilidade.
- **Confiabilidade transacional (ADR-003):** cada serviço usa transações ACID no próprio schema. O saldo de créditos fica consistente sem sagas nem transações distribuídas, que os microsserviços exigiriam.
- **Escalabilidade da captura (ADR-004):** a captura é um serviço separado, alimentado por fila, que escala sozinho.

A escalabilidade da captura é a razão técnica mais forte para não usar um monólito. A confiabilidade transacional e a viabilidade são as razões mais fortes para não usar microsserviços. O service-based fica no meio: traz a modularidade, o isolamento de falhas e a escala seletiva dos serviços com uma fração do custo operacional dos microsserviços.

A granularidade grossa também combina com o domínio: seis serviços correspondem a seis áreas de negócio bem definidas no documento de requisitos, cada uma com um conjunto claro de requisitos.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Escala seletiva da captura e isolamento de falhas | **Mais componentes que um monólito:** Gateway, fila, storage e seis serviços para implantar, monitorar e testar |
| Transações ACID locais em cada domínio | **Banco compartilhado como ponto único de falha e possível gargalo** quando houver muitos workers (ADR-004, ADR-005) |
| Implantação independente por serviço | **Comunicação pela rede:** chamadas entre serviços são mais lentas e podem falhar, o que exige timeouts e disjuntores (ADR-005) |
| Custo e complexidade compatíveis com a equipe | **Escalabilidade limitada fora da captura:** serviços de granularidade grossa escalam como um todo |
| Caminho aberto para separar um serviço em microsserviço no futuro | **Disciplina obrigatória:** sem a regra de acesso por schema (ADR-009), o banco compartilhado acopla os serviços e anula o benefício do estilo |

## Consequências

**Positivas**
- A arquitetura atende às características prioritárias e cabe no prazo e na equipe.
- Os diagramas de componentes e de pacotes seguem diretamente da divisão em seis serviços e da estrutura em camadas de cada um (ADR-009).
- Se no futuro a captura ou os pagamentos precisarem de mais independência, podem ganhar banco próprio e virar microsserviços sem reescrever o resto.

**Negativas / riscos**
- O banco compartilhado precisa ser monitorado desde o início; sua replicação ou divisão exigirá uma nova ADR.
- A equipe precisa dominar comunicação assíncrona, idempotência e outbox, que são menos triviais que um CRUD em monólito.
- Se o prazo de implementação apertar, a migração para o plano B (monólito modular) precisará de uma nova ADR que substitua esta.
