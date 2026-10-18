# Componentes candidatos do Veridit

Lista dos componentes candidatos da arquitetura, com as responsabilidades de cada um segundo os requisitos ([`veridit-req.pdf`](../veridit-req.pdf)) e as características arquiteturais ([ADRs](adr/)).

O sistema segue o estilo service-based ([ADR-011](adr/ADR-011-estilo-arquitetural-service-based.md)): seis serviços de domínio, um banco PostgreSQL com um schema por serviço, fila (RabbitMQ) para as operações demoradas e object storage (MinIO) para os arquivos. São **21 componentes**: 15 de domínio, distribuídos nos seis serviços, e 6 de infraestrutura e apoio.

Os códigos C01 a C21 são os mesmos usados no diagrama de componentes e no diagrama de pacotes.

## Componentes de domínio

### Interface Web + API Gateway

| Código | Componente | Responsabilidade | Requisitos | Características |
|---|---|---|---|---|
| C01 | Aplicação Web | Telas do usuário e da área administrativa, servidas como página estática com cache. Mostra o status das operações demoradas por consulta periódica. | Todos, D2 | Desempenho |
| C02 | API Gateway | Única entrada das APIs: roteamento, validação local do token, rotas públicas explícitas e limite de tentativas. Liga o navegador do usuário ao worker de vídeo (proxy de sessão). Roda com duas instâncias. | Todos | Segurança, disponibilidade |

### Identidade

| Código | Componente | Responsabilidade | Requisitos | Características |
|---|---|---|---|---|
| C03 | Contas e Perfis | Cadastro com conta base e perfil (`Advogado`, `UsuarioComum` e, no futuro, `Empresa`), validação do CPF, confirmação de e-mail e CPF criptografado. Atende os direitos do titular e o encerramento da conta. Fornece nome e CPF ao relatório por API interna. | REQ01, REQ14, D1 | LGPD, extensibilidade |
| C04 | Autenticação | Login, emissão de JWT e token de renovação, logout com revogação, hash de senha (Argon2), recuperação de senha com token de uso único e publicação da chave pública de verificação. | REQ02, REQ03, REQ04 | Segurança |

### Créditos e Pagamentos

| Código | Componente | Responsabilidade | Requisitos | Características |
|---|---|---|---|---|
| C05 | Catálogo e Pedidos | Pacotes de créditos como dados, gerenciados pela área administrativa. Pedidos de compra com preço congelado e dados de faturamento no schema isolado. | REQ05, D2 | LGPD, extensibilidade |
| C06 | Integração de Pagamentos | Adaptador do Mercado Pago (checkout e Pix). Recebe webhooks validando a assinatura, consultando o provedor e evitando duplicidade. Reconcilia periodicamente os pagamentos pendentes. | REQ06 | Confiabilidade transacional, interoperabilidade |
| C07 | Livro-razão de Créditos | Lançamentos imutáveis de compra, reserva, consumo, devolução e estorno, cálculo do saldo e regra de saldo negativo. Publica `PagamentoAprovado` e `CreditoDevolvido` pela outbox. | REQ05, REQ07, REQ08 | Confiabilidade transacional |

### Captura

| Código | Componente | Responsabilidade | Requisitos | Características |
|---|---|---|---|---|
| C08 | API de Captura | Inicia o registro, reserva o crédito, enfileira a captura de tela ou aloca o navegador de vídeo, controla os estados do registro e expira os parados. Ao receber `CapturaExecutada`, confere os hashes, pede a assinatura do manifesto, marca `CONCLUIDO` e publica `CapturaConcluida`. Executa o expurgo controlado e publica `EvidenciaExpurgada`. | REQ08, REQ09, REQ11 | Integridade, disponibilidade, LGPD |
| C09 | Workers de Captura de Tela | Consomem a fila e fazem cada captura num navegador novo, usando o Gerador de Evidência (C11). Escalam pelo tamanho da fila. | REQ08, REQ11 | Escalabilidade |
| C10 | Pool de Workers de Vídeo | Mantém containers limpos já iniciados, de uso único, com limite de sessões simultâneas. Transmite a tela para o usuário durante a navegação e usa o Gerador de Evidência (C11). | REQ08, REQ09 | Escalabilidade |
| C11 | Gerador de Evidência | Módulo comum aos workers: coleta os metadados técnicos (IP, DNS, certificado TLS, cabeçalhos HTTP, HTML), calcula os hashes, monta o manifesto, grava no storage, confere a gravação e publica `CapturaExecutada`. | REQ08, REQ09, REQ11 | Integridade |

### Registros e Relatórios

| Código | Componente | Responsabilidade | Requisitos | Características |
|---|---|---|---|---|
| C12 | Consulta e Download de Registros | Listagem paginada e detalhes dos registros, lidos de uma cópia mantida pelos eventos da Captura. Gera links de download assinados, válidos por poucos minutos, depois de conferir o dono. | REQ12, REQ13, REQ14, REQ15 | Desempenho |
| C13 | Geração de Relatórios e Zips | Workers que geram, uma única vez, o relatório (campos do Apêndice A, manifesto, assinatura e hashes) e o zip, guardados no bucket de arquivos derivados. | REQ14, REQ15 | Extensibilidade, desempenho |
| C14 | Trilha de Auditoria | Grava, somente por inclusão, o evento `EventoAuditoria` publicado pelos serviços: início, conclusão e falha de capturas, ações no vídeo, geração de relatório e de link de download, e expurgos. | REQ08 a REQ15 | Integridade |

### Notificação

| Código | Componente | Responsabilidade | Requisitos | Características |
|---|---|---|---|---|
| C15 | Notificação | Transforma `PagamentoAprovado` e `CapturaConcluida` em e-mails (o de conclusão leva o hash do manifesto) e os envia por SMTP, com novas tentativas. | REQ07, REQ10, REQ11 | Interoperabilidade |

## Componentes de infraestrutura e apoio

| Código | Componente | Responsabilidade | Características |
|---|---|---|---|
| C16 | Mensageria (RabbitMQ) | Filas duráveis com confirmação manual e fila de mensagens mortas para os fluxos assíncronos. | Escalabilidade, disponibilidade |
| C17 | Banco de Dados e Backup | PostgreSQL com um schema e um usuário de banco por serviço. Arquivamento contínuo, backup diário e testes mensais de restauração. | Confiabilidade transacional, durabilidade |
| C18 | Armazenamento de Evidências | MinIO com bucket imutável das evidências, bucket de arquivos derivados e cópia externa. Tarefa periódica que confere os hashes contra o manifesto. | Integridade, durabilidade |
| C19 | Cofre de Segredos e Chaves | Guarda credenciais de serviço, segredos e a chave de assinatura dos manifestos. | Segurança |
| C20 | Ambiente Isolado de Captura | Containers sem privilégios e descartáveis, rede com saída só para a internet, bloqueio de endereços internos e relógio sincronizado por NTP. | Segurança, integridade |
| C21 | Monitoramento e Alertas | Acompanha disponibilidade, filas, fila de mensagens mortas, atraso da cópia externa e relógio dos workers. | Disponibilidade |

## Componentes críticos com detalhamento interno

Os 7 componentes abaixo (33% do total) serão detalhados internamente no diagrama de componentes. Cada um sustenta diretamente uma das quatro características prioritárias (ADR-001 a ADR-004), e uma falha em qualquer deles inviabiliza o produto: expõe dados, perde dinheiro do cliente ou compromete o valor da prova.

| Código | Componente | Partes internas | Característica prioritária |
|---|---|---|---|
| C02 | API Gateway | Roteamento, validação de token, limite de tentativas, proxy de sessão de vídeo | Segurança |
| C04 | Autenticação | Login, emissão e revogação de tokens, recuperação de senha, hash de senha | Segurança |
| C06 | Integração de Pagamentos | Adaptador do Mercado Pago, receptor de webhooks, reconciliação | Confiabilidade transacional |
| C07 | Livro-razão de Créditos | Lançamentos, cálculo de saldo, reserva e devolução, outbox | Confiabilidade transacional |
| C08 | API de Captura | Início e reserva de crédito, máquina de estados, expiração, conferência e assinatura, expurgo | Integridade, escalabilidade |
| C10 | Pool de Workers de Vídeo | Alocação de sessão, navegador remoto, transmissão de tela | Escalabilidade, integridade |
| C11 | Gerador de Evidência | Coletor de metadados, cálculo de hashes, montagem do manifesto, gravação e conferência | Integridade |

## Principais interações

- **Compra de créditos:** C01 → C02 → C05 (pedido) → C06 (cobrança no Mercado Pago). O webhook chega em C06, que confirma com o provedor; C07 lança os créditos e publica `PagamentoAprovado`; C15 envia o e-mail.
- **Captura de tela:** C01 → C02 → C08 (cria o registro e reserva o crédito em C07) → C16 (fila) → C09 com C11 (captura, manifesto e `CapturaExecutada`) → C08 (confere, assina com C19, publica `CapturaConcluida`) → C12, C13, C14 e C15 reagem ao evento; C07 converte a reserva em consumo.
- **Vídeo de navegação:** igual à captura de tela, mas C08 aloca um navegador em C10, e o usuário navega pelo proxy de sessão de C02.
- **Consulta e download:** C01 → C02 → C12 (lista, detalha e gera o link) → download direto de C18.

## Rastreabilidade dos requisitos

| Requisito | Componentes |
|---|---|
| REQ01 Cadastrar usuário | C03 |
| REQ02 Recuperar senha | C04 |
| REQ03 e REQ04 Login e logout | C04, C02 |
| REQ05 Comprar créditos | C05, C07 |
| REQ06 Efetuar pagamento | C06 |
| REQ07 Confirmar pagamento por e-mail | C07, C15 |
| REQ08 Iniciar registro de conteúdo | C08, C09, C10, C11 |
| REQ09 Concluir gravação | C08, C10, C11 |
| REQ10 Confirmar gravação por e-mail | C15 |
| REQ11 Concluir captura de tela e confirmar por e-mail | C08, C09, C11, C15 |
| REQ12 e REQ13 Listar e detalhar registros | C12 |
| REQ14 Gerar relatório | C13, C12, C03 |
| REQ15 Gerar zip | C13, C12 |
| D1 Tipos de usuário | C03 |
| D2 Pacotes de créditos | C05, C01 |

> As decisões tomadas no agrupamento estão registradas nas ADRs: trilha de auditoria no Serviço de Registros e Relatórios ([ADR-002](adr/ADR-002-integridade-auditabilidade-evidencias.md), item 11), expurgo executado pela API de Captura ([ADR-007](adr/ADR-007-conformidade-lgpd.md), item 6) e área administrativa como rotas em cada serviço ([ADR-009](adr/ADR-009-extensibilidade-manutenibilidade.md), item 2). Os eventos estão no catálogo da [ADR-008](adr/ADR-008-interoperabilidade.md).
