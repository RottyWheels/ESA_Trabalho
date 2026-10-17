# ADR-008: Interoperabilidade com serviços externos e entre serviços

- **Status:** Aceito
- **Data:** 17/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Elaborado com apoio de IA fora de sala, com autorização excepcional do professor; revisado e aprovado pela equipe
- **Revisão 1 (17/10/2026):** Correções de coerência com as demais ADRs: impacto da queda do Mercado Pago alinhado à ADR-005 (compras ficam indisponíveis). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 2 (17/10/2026):** Checklist de revisão (D1: cabeçalho). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 3 (17/10/2026):** Coerência com as demais ADRs: catálogo de eventos com produtor e consumidores (item 6) e evento `PagamentoEstornado` na camada anticorrupção (item 2). Feita com apoio de IA fora de sala.

## Contexto

O Veridit depende de integrações externas em quase todos os fluxos principais:

- **Pagamento:** Mercado Pago e Pix (REQ06). O Veridit não gera Pix sozinho: a cobrança Pix (QR Code ou chave copia e cola) precisa ser criada por um provedor de serviços de pagamento (PSP), que pode ser o próprio Mercado Pago ou um banco. A confirmação chega por webhook.
- **E-mail:** confirmações de pagamento e de captura (REQ07, REQ10, REQ11).
- **Captura:** acesso a sites de terceiros por um navegador no servidor (REQ08).
- **Armazenamento:** object storage para as evidências (ADR-002, ADR-006).

Cada provedor tem sua própria API, formato de dados e regras, e pode mudar, encarecer ou sair do ar. Internamente, os seis serviços também precisam trocar dados entre si por APIs e eventos.

## Requisitos e restrições relacionados

- **REQ06:** Pagamento via Mercado Pago ou Pix
- **REQ07, REQ10 e REQ11:** E-mails de confirmação
- **REQ08:** Captura de sites de terceiros
- **D2:** Novos pacotes de créditos, que podem trazer novos meios de pagamento
- **ADR-003:** Webhooks idempotentes e consulta ativa ao provedor
- **ADR-005:** Timeouts, disjuntores e novas tentativas nas integrações
- **ADR-006:** Compatibilidade com a API S3 para migrar o storage

## Decisão

Tratar **interoperabilidade** com as seguintes táticas:

1. **Portas e adaptadores.** A lógica de negócio de cada serviço depende apenas de interfaces internas (portas). Cada provedor é implementado em um adaptador separado:

   | Porta | Adaptadores na primeira versão | Serviço |
   |---|---|---|
   | `ProvedorPagamento` | Mercado Pago (checkout e Pix) | Créditos e Pagamentos |
   | `EnvioEmail` | SMTP genérico | Notificação |
   | `NavegadorCaptura` | Playwright com Chromium | Captura |
   | `ArmazenamentoEvidencias` | API S3 (MinIO) | Captura, Registros e Relatórios |

2. **Camada anticorrupção para webhooks.** Cada provedor de pagamento tem um endpoint de webhook próprio, que valida a assinatura, consulta o provedor (ADR-003) e traduz a notificação para um evento interno único (`PagamentoAprovado`, `PagamentoRecusado`, `PagamentoExpirado`, `PagamentoEstornado`). O restante do sistema não conhece os formatos do Mercado Pago.
3. **Um provedor de Pix na primeira versão.** O Pix é gerado pelo Mercado Pago, o mesmo provedor do checkout, para ter um único contrato, um único webhook e uma única reconciliação. Outro PSP pode ser adicionado depois como novo adaptador.
4. **Protocolos e padrões abertos:** API S3 para storage, SMTP para e-mail, HTTP com JSON para APIs, JWT para autenticação (pronto para OpenID Connect no futuro, ADR-001).
5. **Contratos das APIs internas** documentados em OpenAPI, com versão no caminho (`/v1/...`). Mudanças incompatíveis exigem uma nova versão, mantida em paralelo até todos os consumidores migrarem.
6. **Eventos com esquema versionado.** Cada mensagem na fila tem tipo, versão do esquema, id único (para idempotência) e data/hora. Consumidores ignoram campos que não conhecem. Os eventos da primeira versão são estes; um evento novo entra nesta tabela antes de ser usado:

   | Evento | Produtor | Consumidores |
   |---|---|---|
   | `PagamentoAprovado` | Créditos e Pagamentos (outbox, ADR-003) | Notificação (e-mail de confirmação, REQ07) |
   | `PagamentoRecusado`, `PagamentoExpirado`, `PagamentoEstornado` | Camada anticorrupção de Créditos e Pagamentos | O próprio Serviço de Créditos e Pagamentos (máquina de estados e livro-razão, ADR-003) |
   | `CreditoDevolvido` | Créditos e Pagamentos (outbox, ADR-003) | Nenhum obrigatório na primeira versão; fica disponível para aviso ao usuário |
   | `RegistroCriado` | API de Captura (outbox) | Registros e Relatórios (cópia de leitura, ADR-009) |
   | `CapturaConcluida` | API de Captura (outbox) | Registros e Relatórios (cópia de leitura e geração de relatório e zip, ADR-010), Créditos e Pagamentos (consumo da reserva, ADR-003), Notificação (e-mail com o hash do manifesto, REQ10, REQ11, ADR-002) |
   | `CapturaFalhou`, `CapturaCancelada` | API de Captura (outbox) | Registros e Relatórios (cópia de leitura), Créditos e Pagamentos (devolução da reserva, ADR-003) |
   | `CapturaExecutada`, `CapturaComErro` | Worker de captura | Só a API de Captura (ADR-002) |
   | Pedido de início da captura de tela (pedido e resposta) | Worker de captura | Só a API de Captura, que responde se o worker pode começar (ADR-002) |
7. **Configuração externa.** Credenciais, URLs e escolha do adaptador ficam em variáveis de ambiente ou no cofre de segredos (ADR-001), não no código.
8. **Testes de contrato** contra os ambientes de teste (sandbox) dos provedores e entre os serviços internos, executados na integração contínua.

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Troca de provedor de e-mail | Feita só com um novo adaptador e configuração, sem alterar a lógica de negócio |
| Código de negócio que importa SDK de provedor externo | Zero ocorrências (verificado por regra automática de dependências) |
| APIs internas com contrato OpenAPI | 100% |
| Integrações externas com teste de contrato em sandbox | 100% |

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Chamar os SDKs dos provedores direto na lógica de negócio | Mais rápido de escrever, mas espalha a dependência pelo código e torna a troca de provedor uma reescrita. |
| Barramento de integração (ESB) ou plataforma de integração (iPaaS) | Centraliza as integrações, mas adiciona um componente caro e complexo para quatro integrações. |
| gRPC entre os serviços internos | Mais eficiente, mas exige ferramentas extras, é menos prático para depurar e não serve à interface web sem tradução. HTTP com JSON atende ao volume previsto. |
| Pix por um PSP diferente do checkout desde a primeira versão | Pode ter tarifas menores, mas dobra contratos, webhooks e reconciliação. Fica como evolução, via novo adaptador. |

## Justificativa

A interoperabilidade ficou fora do top 4 porque é resolvida com padrões de projeto bem conhecidos e não muda a divisão em serviços. Mas ela protege as quatro características prioritárias: o pagamento confiável (ADR-003) depende de isolar o formato do Mercado Pago, a escalabilidade da captura (ADR-004) e a durabilidade (ADR-006) dependem de poder trocar navegador e storage.

Esta ADR também resolve uma ambiguidade dos requisitos: o REQ06 fala em gerar Pix "pela plataforma do Veridit", mas isso só é possível por meio de um PSP. Usar o próprio Mercado Pago para o Pix simplifica a primeira versão sem impedir outro provedor no futuro.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Troca ou inclusão de provedores sem mexer nas regras de negócio | **Mais código:** interfaces, adaptadores e tradutores para cada integração |
| Formatos externos não vazam para o resto do sistema | **Menor aproveitamento:** recursos específicos de um provedor ficam escondidos atrás da interface comum |
| APIs e eventos documentados e versionados | **Manutenção:** contratos e versões paralelas precisam ser mantidos e testados |
| Um único provedor de pagamento simplifica a primeira versão | **Dependência:** concentrar checkout e Pix no Mercado Pago aumenta o impacto se ele ficar fora do ar: as compras param até ele voltar (ADR-005) |

## Consequências

**Positivas**
- Novos meios de pagamento, provedores de e-mail ou storage em nuvem entram como novos adaptadores.
- A lógica de negócio pode ser testada sem os provedores reais, usando adaptadores falsos.
- Os contratos OpenAPI servem de documentação para as etapas de implementação e evolução.

**Negativas / riscos**
- Com Pix e checkout no mesmo provedor, não há meio de pagamento alternativo: uma queda do Mercado Pago impede compras, e a ADR-005 trata isso como degradação controlada. Um segundo provedor de Pix é a evolução prevista.
- Os ambientes de teste dos provedores nem sempre reproduzem o comportamento real dos webhooks.
