# Tecnologias do Veridit

Mapeamento das tecnologias necessárias para implementar a arquitetura descrita nas [ADRs](adr/) e nos [componentes candidatos](componentes.md). Por ser um trabalho universitário, todas as tecnologias são **gratuitas**: código aberto ou plano gratuito.

As ADRs já definem a infraestrutura: PostgreSQL, RabbitMQ, MinIO, Playwright com Chromium, Docker Compose ([ADR-011](adr/ADR-011-estilo-arquitetural-service-based.md)), JWT e Argon2 ([ADR-001](adr/ADR-001-seguranca.md)), SMTP, OpenAPI e Mercado Pago ([ADR-008](adr/ADR-008-interoperabilidade.md)). A linguagem e os frameworks abaixo são uma **proposta** e ainda precisam ser registrados em ADR, em sala.

## Linguagem

**TypeScript (Node.js) em todo o sistema**, do frontend aos workers:

- O Playwright é nativo de Node.js, e a captura (C09 a C11) é a parte mais difícil do sistema.
- Uma única linguagem permite que qualquer membro da equipe trabalhe em qualquer serviço ([ADR-009](adr/ADR-009-extensibilidade-manutenibilidade.md)).

Alternativa: Python com FastAPI, também gratuita e com Playwright disponível, mas com o frontend em outra linguagem.

## Mapeamento por camada

| Camada | Tecnologia | Para quê | Componentes |
|---|---|---|---|
| Frontend | React + Vite | Telas do usuário e da área administrativa | C01 |
| | MUI ou shadcn/ui | Componentes visuais prontos | C01 |
| Gateway | Fastify + `@fastify/http-proxy` | Roteamento, validação do JWT e proxy WebSocket do vídeo | C02 |
| | `@fastify/rate-limit` | Limite de tentativas de login e de recuperação de senha ([ADR-001](adr/ADR-001-seguranca.md)) | C02 |
| | nginx (ou Traefik) | Terminação TLS e balanceamento entre as duas instâncias do Gateway e da Identidade ([ADR-005](adr/ADR-005-disponibilidade-tolerancia-falhas.md)) | C02, C04 |
| Serviços | NestJS (ou Fastify puro) | Estrutura em camadas (`api`, `aplicacao`, `dominio`, `infraestrutura`). Os workers de captura são processos Node.js simples que consomem a fila | C03 a C08, C12 a C15 |
| | Zod | Validação de entrada | Todos |
| | `@nestjs/swagger` | Contratos OpenAPI ([ADR-008](adr/ADR-008-interoperabilidade.md)) | Todos |
| | `opossum` | Timeouts e disjuntor nas chamadas síncronas ([ADR-005](adr/ADR-005-disponibilidade-tolerancia-falhas.md)) | C02, C06, C08, C12, C13 |
| | `@nestjs/schedule` | Tarefas periódicas: outbox, expiração de reservas presas ([ADR-005](adr/ADR-005-disponibilidade-tolerancia-falhas.md)), verificação de integridade ([ADR-006](adr/ADR-006-durabilidade-recuperacao-evidencias.md)) e expurgo ([ADR-007](adr/ADR-007-conformidade-lgpd.md)) | C07, C08, C13 |
| Segurança | `jose` | Emissão e validação de JWT (biblioteca comum, [ADR-009](adr/ADR-009-extensibilidade-manutenibilidade.md)) | C02, C04, todos |
| | `argon2` | Hash de senha | C04 |
| | `node:crypto` | SHA-256, assinatura Ed25519 do manifesto e HMAC do CPF | C03, C08, C11 |
| Banco | PostgreSQL 16 | Um schema e um usuário de banco por serviço; trilha de auditoria em tabela só de inclusão | C17, C14 |
| | Prisma (ou Drizzle) | ORM e migrações por schema | Todos |
| | pgBackRest (ou WAL-G) | Arquivamento contínuo do WAL e backup diário criptografado ([ADR-006](adr/ADR-006-durabilidade-recuperacao-evidencias.md)) | C17 |
| Mensageria | RabbitMQ (com painel web) | Filas duráveis, `ack` manual e fila de mensagens mortas | C16 |
| | `amqplib` | Cliente do RabbitMQ | Todos |
| | Tabela de outbox + tarefa periódica | Outbox transacional, sem biblioteca extra | C07, C08 |
| Armazenamento | MinIO (nó único no MVP, ver pontos de atenção) | Buckets de evidências e de arquivos derivados, com versionamento e bloqueio de escrita (Object Lock, [ADR-002](adr/ADR-002-integridade-auditabilidade-evidencias.md)) | C18 |
| | `mc mirror` ou replicação de buckets do MinIO | Cópia das evidências em um segundo destino compatível com S3 ([ADR-006](adr/ADR-006-durabilidade-recuperacao-evidencias.md)) | C18 |
| | `@aws-sdk/client-s3` | Cliente compatível com S3 ([ADR-006](adr/ADR-006-durabilidade-recuperacao-evidencias.md), [ADR-008](adr/ADR-008-interoperabilidade.md)) | C11, C12, C13 |
| Captura | Playwright + Chromium | Captura de tela, gravação de vídeo (`recordVideo`) e metadados (TLS, cabeçalhos, HTML) | C09, C10, C11 |
| | CDP `Page.startScreencast` + `ws` | Transmissão do vídeo de navegação ao vivo | C10 |
| Ambiente isolado | Rede Docker separada + Smokescreen | Proxy de saída dos navegadores: bloqueia endereços privados, de loopback e de metadados depois da resolução de DNS e de cada redirecionamento ([ADR-001](adr/ADR-001-seguranca.md)) | C20 |
| | Opções de segurança do Docker | Usuário não root, `cap_drop: ALL`, `no-new-privileges`, sistema de arquivos só leitura e limites de CPU e memória ([ADR-001](adr/ADR-001-seguranca.md), [ADR-004](adr/ADR-004-escalabilidade-elasticidade-captura.md), [ADR-005](adr/ADR-005-disponibilidade-tolerancia-falhas.md)) | C20 |
| | chrony (NTP) | Relógio dos workers sincronizado, com horários gravados em UTC ([ADR-002](adr/ADR-002-integridade-auditabilidade-evidencias.md)) | C20 |
| Relatórios | Playwright `page.pdf()` | PDF do relatório a partir de um template HTML | C13 |
| | `archiver` | Geração do zip | C13 |
| E-mail | Nodemailer | Envio por SMTP | C15 |
| | Mailpit | Servidor SMTP de teste com caixa de entrada web | C15 |
| Pagamento | SDK `mercadopago` + ambiente de teste do Mercado Pago | Checkout, Pix e webhooks, sem cobrança real | C06 |
| | Cloudflare Tunnel (ou ngrok, plano gratuito) | URL pública para o webhook chegar à máquina local | C06 |
| Segredos | Variáveis de ambiente + `.env` fora do Git | Substitui o cofre no MVP | C19 |
| | OpenBao | Cofre de segredos livre (evolução) | C19 |
| Execução | Docker + Docker Compose | Sobe o sistema inteiro com um comando ([ADR-011](adr/ADR-011-estilo-arquitetural-service-based.md)) | Todos |
| Testes | Vitest (ou Jest) | Testes de unidade do domínio | Todos |
| | Testcontainers | Testes de integração com Postgres, RabbitMQ e MinIO reais | Todos |
| | Playwright Test | Testes de ponta a ponta da interface | C01 |
| | k6 | Testes de carga ([ADR-010](adr/ADR-010-desempenho.md)) | — |
| Qualidade e CI | ESLint + Prettier | Padrão de código | Todos |
| | GitHub Actions | Pipeline de integração contínua ([ADR-009](adr/ADR-009-extensibilidade-manutenibilidade.md)) | — |
| Monitoramento | Prometheus + Alertmanager | No MVP, só os alertas exigidos pelas ADRs: fila de mensagens mortas (plugin Prometheus do RabbitMQ, [ADR-005](adr/ADR-005-disponibilidade-tolerancia-falhas.md)) e diferença do relógio dos workers (`node_exporter`, [ADR-002](adr/ADR-002-integridade-auditabilidade-evidencias.md)) | C21 |
| | Grafana | Painéis de acompanhamento (evolução, fora do MVP) | C21 |
| Documentação | draw.io (diagrams.net) ou PlantUML | Diagramas de componentes e de pacotes UML | — |
| | Figma (plano gratuito ou de educação) | Telas da aplicação | — |

## Pontos de atenção

- **Mercado Pago:** o ambiente de teste é gratuito, mas exige conta e usuários de teste. Verificar cedo como o Pix se comporta no sandbox; se não houver confirmação simulada, usar um adaptador falso na demonstração ([ADR-008](adr/ADR-008-interoperabilidade.md)).
- **GitHub Actions:** ilimitado em repositório público; em repositório privado há uma cota mensal de minutos gratuitos.
- **GitHub Student Developer Pack:** vale ativar com o e-mail institucional da UFBA, que amplia cotas e dá acesso a outras ferramentas.
- **Cofre de segredos:** o HashiCorp Vault deixou de ser código aberto; por isso a indicação é o OpenBao, um fork livre.
- **MinIO de nó único (corte do MVP):** a [ADR-006](adr/ADR-006-durabilidade-recuperacao-evidencias.md) (item 2) prevê MinIO em modo distribuído com *erasure coding*. No MVP ele roda num nó só, para caber no prazo e numa máquina local. Esse corte **precisa ser registrado na ADR-006, em sala**. Para não perder versionamento e Object Lock (ADR-002), o nó único deve usar vários volumes (pelo menos 4), já que essas funções exigem *erasure coding* no MinIO; assim ele tolera a perda de discos, mas não do servidor. Confirmar isso na documentação do MinIO ao montar o ambiente.
- **Hospedagem:** não é necessária. O sistema roda localmente pelo Docker Compose, e a única peça exposta à internet é o túnel do webhook.
