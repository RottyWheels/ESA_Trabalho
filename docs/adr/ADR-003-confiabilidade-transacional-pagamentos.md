# ADR-003: Confiabilidade transacional de pagamentos e créditos como característica arquitetural prioritária

- **Status:** Aceito
- **Data:** 13/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Rascunho elaborado com apoio de IA em sala, sujeito à revisão e aprovação da equipe
- **Revisão 1 (17/10/2026):** Pix por meio de provedor de serviços de pagamento, detalhes da reserva de créditos (chamada síncrona, idempotência e expiração), regra para estorno de pagamento com créditos já usados e metas de verificação. Revisão feita com apoio de IA fora de sala, com autorização excepcional do professor.

## Contexto

O modelo de negócio do Veridit é por **créditos pré-pagos**. O usuário escolhe um pacote (básico, médio ou premium), informa os dados de faturamento e paga pelo Mercado Pago ou por Pix gerado na própria plataforma. O Veridit não gera Pix sozinho: a cobrança Pix é criada por um provedor de serviços de pagamento, que na primeira versão é o próprio Mercado Pago (ADR-008). Depois, consome créditos ao fazer registros de conteúdo.

O pagamento é **assíncrono e externo**. A confirmação chega por notificação do provedor (webhook), que pode atrasar, chegar duplicada, fora de ordem ou nem chegar. O Pix pode expirar sem ser pago. Do outro lado, a captura pode falhar no meio por timeout, site fora do ar ou queda do worker.

Na arquitetura service-based, o saldo fica no Serviço de Créditos e Pagamentos, mas é consumido por um fluxo iniciado no Serviço de Captura e confirmado ao usuário pelo Serviço de Notificação. Ou seja, a operação atravessa serviços.

## Requisitos e restrições relacionados

- **REQ05:** Comprar créditos (selecionar pacote, informar faturamento, pagar)
- **REQ06:** Efetuar pagamento via Mercado Pago ou Pix (QR Code ou chave)
- **REQ07:** Confirmar pagamento por e-mail
- **REQ08:** Iniciar registro de conteúdo, que consome créditos
- **D2:** Pacotes básico, médio e premium, com previsão de novos pacotes
- **Apêndice A:** Pacote (quantidade de créditos, valor por crédito) e dados da compra
- **Restrição externa:** API e webhooks do Mercado Pago e do Pix
- **Lacuna do documento:** não diz quantos créditos cada registro consome nem o que acontece com o crédito em caso de falha. Este ADR adota a premissa de devolver o crédito quando a captura falha.

## Decisão

Adotar **Confiabilidade transacional de pagamentos e créditos** como característica arquitetural prioritária (3ª do top 4), implementada pelos seguintes mecanismos:

1. **Livro-razão de créditos somente de inclusão.** Cada movimentação (compra, reserva, consumo, estorno) é um lançamento imutável. O saldo é calculado a partir dos lançamentos ou mantido em cache dentro da mesma transação.
2. **Máquina de estados do pagamento:** `PENDENTE` → `APROVADO` | `RECUSADO` | `EXPIRADO`, e `APROVADO` → `ESTORNADO`. Transições inválidas são rejeitadas.
3. **Idempotência nos webhooks.** O id do pagamento no Mercado Pago ou o identificador da transação Pix (txid) é chave única. Receber a mesma notificação de novo não gera crédito em dobro.
4. **Validação da origem do webhook** (assinatura ou segredo) e **consulta ativa ao provedor** antes de aprovar. O sistema nunca confia apenas no conteúdo da notificação nem no redirecionamento do navegador.
5. **Reconciliação periódica.** Uma tarefa agendada consulta no provedor os pagamentos `PENDENTE` antigos, para recuperar webhooks perdidos e marcar Pix expirados.
6. **Reserva → consumo → estorno.** Ao iniciar o registro (REQ08), o crédito é reservado. Ao concluir (REQ09, REQ11), a reserva vira consumo. Se a captura falhar, a reserva é desfeita e o crédito volta.
   - A reserva é uma chamada síncrona do Serviço de Captura ao de Créditos, feita antes de o trabalho entrar na fila. Se Créditos estiver fora do ar, a captura não começa e o usuário recebe uma mensagem clara (ADR-005).
   - Reserva, consumo e devolução são idempotentes pelo id do registro: repetir a mesma operação não reserva nem devolve duas vezes.
   - Reservas que não forem concluídas dentro do tempo máximo da captura são desfeitas automaticamente, para que a queda de um worker não prenda créditos (ADR-005).
7. **Outbox transacional.** Os eventos `PagamentoConfirmado` e `CreditoEstornado` são gravados na mesma transação da mudança de estado e só depois publicados na fila. Isso garante o e-mail de confirmação (REQ07) sem perder nem duplicar eventos.
8. **Preço congelado na compra.** O valor e a quantidade de créditos do pacote são copiados para o pedido no momento da compra, para que mudanças futuras nos pacotes (D2) não alterem compras já feitas.
9. **Estorno pelo provedor com créditos já usados.** Se um pagamento aprovado for estornado ou contestado no provedor, os créditos daquela compra são debitados no livro-razão. Se já tiverem sido usados, o saldo fica negativo e novas capturas ficam bloqueadas até a regularização. Os registros já feitos continuam acessíveis ao usuário.

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Webhook repetido (mesmo id de pagamento ou txid) | Nenhum crédito em dobro em teste com 1.000 repetições |
| Webhook perdido | Pagamento aprovado reconhecido pela reconciliação em até 30 minutos |
| Captura que falha | Crédito devolvido em 100% dos casos |
| Reserva presa por queda de worker | Desfeita automaticamente após o tempo máximo da captura |
| Saldo calculado pelo livro-razão | Igual ao saldo em cache em 100% das verificações diárias |

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Campo `saldo` atualizado diretamente (somar e subtrair) | Mais simples, mas não deixa histórico, fica sujeito a erros de concorrência e não permite reconstruir ou auditar o saldo. |
| Confirmar o pagamento pelo redirecionamento do checkout | O navegador do usuário não é fonte confiável: o redirecionamento pode ser forjado ou nem acontecer. |
| Transação distribuída entre serviços (2PC) | Garante atomicidade, mas acopla os serviços, reduz a disponibilidade e tem pouco suporte em brokers e provedores externos. |
| Debitar o crédito só ao concluir a captura | Evita estorno, mas permite iniciar várias capturas com saldo para uma só e consumir recursos sem garantia de pagamento. |
| Confiabilidade fora do top 4 | Erros financeiros afetam diretamente a confiança do cliente e geram reclamações e estornos. São inaceitáveis em um produto pago. |

## Justificativa

Confiabilidade transacional entra no top 4 porque o fluxo de créditos é o **coração do modelo de negócio** e envolve **dinheiro do cliente**. Um erro aqui, seja crédito em dobro, crédito perdido ou captura sem saldo, causa prejuízo financeiro ou de reputação na hora, e é difícil de corrigir depois.

A arquitetura service-based ajuda nessa decisão. Os dados financeiros ficam em um único serviço com schema próprio, onde valem transações ACID locais. A comunicação com Captura e Notificação acontece por eventos confiáveis (outbox), sem precisar de transação distribuída.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Correção financeira: nenhum crédito em dobro ou perdido | **Complexidade:** livro-razão, máquina de estados, outbox, idempotência e reconciliação exigem bem mais código que um campo de saldo |
| Histórico completo e auditável das movimentações | **Latência percebida:** o crédito só aparece depois da confirmação do provedor, e o usuário pode esperar segundos ou minutos no Pix |
| Recuperação automática de webhooks perdidos e Pix expirados | **Consistência eventual entre serviços:** o e-mail (REQ07) e a liberação da captura chegam com pequeno atraso em relação ao pagamento |
| Devolução justa do crédito quando a captura falha | **Volume de dados:** o livro-razão só cresce e pode exigir consolidação periódica do saldo |
| Pacotes podem mudar (D2) sem afetar compras antigas | **Testes mais caros:** é preciso simular webhooks duplicados, fora de ordem e perdidos, além de falhas na captura |
| | **Acoplamento ao provedor:** o fluxo depende do comportamento da API do Mercado Pago e do Pix, e pede adaptador para isolar a integração |

## Consequências

**Positivas**
- O saldo do usuário pode ser reconstruído e auditado a qualquer momento.
- Falhas na captura e nos webhooks não geram prejuízo ao cliente nem ao Veridit.
- Novos meios de pagamento podem ser incluídos implementando um novo adaptador, sem alterar o livro-razão.

**Negativas / riscos**
- A equipe precisa dominar idempotência, outbox e máquina de estados, padrões menos triviais que um CRUD.
- A tarefa de reconciliação vira componente crítico e precisa ser monitorada.
- A regra de consumo (créditos por registro, tela ou vídeo) precisa ser validada com o cliente, porque o documento de requisitos não a define.
- A regra de saldo negativo após estorno também é uma proposta da equipe e precisa ser validada com o cliente.
