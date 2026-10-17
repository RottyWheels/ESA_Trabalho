# ADR-005: Disponibilidade e tolerância a falhas

- **Status:** Aceito
- **Data:** 17/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Elaborado com apoio de IA fora de sala, com autorização excepcional do professor; revisado e aprovado pela equipe
- **Revisão 1 (17/10/2026):** Correções de coerência com as demais ADRs: comportamento quando o Mercado Pago cai (Pix depende dele, ADR-008) e prazos de expiração por estado do registro. Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 2 (17/10/2026):** Checklist de revisão (A1: capturas interrompidas não voltam para a fila; A2: conclusão com Créditos fora do ar; A5: `NA_FILA` só na captura de tela; B3: impacto da queda da Identidade; D1: cabeçalho). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 3 (17/10/2026):** Coerência com as demais ADRs: e-mail com provedor fora do ar tratado por novas tentativas e fila de mensagens mortas, não pela outbox; início da captura de tela autorizado pela API de Captura; termo "devolução de créditos" no lugar de "estorno". Feita com apoio de IA fora de sala.
- **Revisão 4 (17/10/2026):** Coerência com as demais ADRs: lista completa das chamadas síncronas com timeout e disjuntor (item 6) e queda de uma instância do Gateway durante a transmissão do vídeo (item 3). Feita com apoio de IA fora de sala.

## Contexto

O Veridit depende de vários componentes que podem falhar de forma independente: os workers de captura (navegador no servidor), o provedor de e-mail, o Mercado Pago e o provedor de Pix, o broker de mensagens, o banco de dados e os próprios sites de terceiros que o usuário quer registrar.

Para o usuário, as funções mais críticas no dia a dia são entrar no sistema, consultar os registros e baixar relatórios e zips, porque é isso que ele leva a um processo. Uma falha na captura ou no e-mail não pode impedir o acesso às provas já registradas.

A ADR-001 também apontou o API Gateway e o Serviço de Identidade como pontos críticos de disponibilidade, e as ADR-003 e ADR-004 já introduziram fila, devolução de créditos e reprocessamento. Esta ADR reúne essas táticas e completa o que falta.

## Requisitos e restrições relacionados

- **REQ03 e REQ04:** Logar e sair do sistema
- **REQ07, REQ10 e REQ11:** E-mails de confirmação, que dependem de provedor externo
- **REQ08 a REQ11:** Captura, que depende de workers e de sites de terceiros
- **REQ12 a REQ15:** Listar, detalhar, gerar relatório e zip, as funções de consulta mais críticas
- **Apêndice A:** Campo "status" dos registros realizados
- **ADR-001:** Gateway e Identidade como pontos críticos
- **ADR-003 e ADR-004:** Fila, reserva e devolução de créditos
- **Restrição do projeto:** Orçamento limitado, sem infraestrutura em várias regiões

## Decisão

Tratar **disponibilidade e tolerância a falhas** com as seguintes táticas:

1. **Isolamento por serviço (bulkhead).** Cada serviço roda em containers próprios, com limites de CPU e memória. Uma falha ou sobrecarga na captura não consome recursos dos demais.
2. **Validação local do token no Gateway.** O Gateway valida o JWT com a chave pública do Serviço de Identidade, sem chamá-lo a cada requisição. Se a Identidade cair, quem já está logado continua usando o sistema; ficam indisponíveis o login, a renovação do token, o cadastro, a confirmação de e-mail e a recuperação de senha. A geração de novos relatórios, que busca nome e CPF na Identidade (ADR-009), espera com novas tentativas até ela voltar.
3. **Gateway e Identidade com pelo menos duas instâncias.** Os dois não guardam estado de sessão em memória, então podem ser replicados atrás de um balanceador. A exceção é a transmissão do vídeo de navegação pelo proxy de sessão do Gateway (ADR-002): a conexão WebSocket fica presa à instância que a recebeu. Se essa instância cair, a sessão de vídeo é interrompida e segue a regra de captura interrompida: o registro vira `FALHOU`, o crédito é devolvido e o usuário refaz o registro.
4. **Mensageria durável.** Filas e mensagens persistentes no RabbitMQ, com confirmação manual (*ack*) só depois do processamento. Se um worker de relatório, zip ou e-mail cai no meio, a mensagem volta para a fila e é processada de novo. Capturas interrompidas não são refeitas: ao receber a mensagem reentregue, o worker pede o início à API de Captura, que recusa porque o registro já não está `NA_FILA` (ADR-002), e a mensagem é descartada. O registro vira `FALHOU` com devolução do crédito (item 8). Uma nova tentativa seria outra captura, em outro momento; o usuário refaz o registro.
5. **Novas tentativas com espera crescente e fila de mensagens mortas.** Falhas temporárias (e-mail, provedor de pagamento, site lento) são repetidas com espera exponencial e variação aleatória. Depois do limite de tentativas, a mensagem vai para uma fila de mensagens mortas, que gera alerta para análise.
6. **Timeouts e disjuntor (*circuit breaker*)** em toda chamada síncrona: Gateway para os serviços, API de Captura para Créditos (reserva) e para o cofre (assinatura do manifesto), Registros e Relatórios para Identidade (dados do relatório), pedido de início do worker à API de Captura pelo broker, e adaptadores dos provedores externos. Um provedor fora do ar falha rápido em vez de prender conexões.
7. **Degradação controlada.** Cada falha tem um comportamento definido:
   - Créditos fora do ar: novas capturas são recusadas com mensagem clara; consulta e download continuam. Capturas já em andamento são concluídas normalmente, e o consumo do crédito é aplicado quando Créditos voltar (ADR-003).
   - Provedor de e-mail fora do ar: o evento já foi publicado pela outbox (ADR-003) e fica na fila do Serviço de Notificação, que repete o envio com espera crescente (item 5). Se o limite de tentativas acabar antes de o provedor voltar, a mensagem vai para a fila de mensagens mortas e é reprocessada quando ele voltar, sem perder o e-mail. O status também aparece na tela.
   - Mercado Pago fora do ar: compras ficam indisponíveis, com mensagem clara, porque o checkout e o Pix dependem dele (ADR-008). Captura, consulta e download continuam com o saldo existente. Um segundo provedor de Pix fica como evolução.
8. **Expiração de reservas presas.** Uma tarefa periódica procura registros parados além do prazo do estado em que estão: um prazo para `NA_FILA` (só na captura de tela) e outro para `EM_ANDAMENTO`, sempre maior que o tempo limite do worker, para que um worker lento mas vivo termine antes. O registro é marcado como `FALHOU` e a reserva de crédito é desfeita (ADR-003). Um worker de captura de tela só começa um trabalho se a API de Captura autorizar a passagem do registro de `NA_FILA` para `EM_ANDAMENTO` (ADR-002). Esta tarefa cobre o worker que morre sem avisar; quando a captura dá erro com o worker vivo, ele publica `CapturaComErro` e o registro vira `FALHOU` na hora.
9. **Verificação de saúde e reinício automático.** Cada serviço expõe verificações de vida e de prontidão. O orquestrador (Docker Compose na v1) reinicia containers que falham.
10. **Monitoramento e alertas** de disponibilidade dos endpoints, tamanho das filas, fila de mensagens mortas e taxa de erro por serviço.

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Login, listagem, detalhes e download | Disponibilidade mensal ≥ 99,5% |
| Início de novas capturas | Disponibilidade mensal ≥ 99% |
| Mensagens de eventos (pagamento, captura, e-mail) | Nenhuma perdida; todas processadas ou na fila de mensagens mortas |
| E-mails de confirmação (REQ07, 10, 11) | 99% entregues ao provedor em até 15 minutos após o evento |
| Queda de um worker de captura | Nenhum impacto em login, listagem e download (teste derrubando o container) |

As metas são iniciais e devem ser revistas depois dos primeiros testes de carga e de falha.

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Implantação ativa em várias regiões ou data centers | Maior disponibilidade, mas custo e complexidade de replicação de dados desproporcionais ao escopo e ao orçamento. Fica como evolução. |
| Comunicação síncrona entre todos os serviços | Mais simples, mas encadeia falhas: um e-mail lento ou um worker travado bloqueia a requisição do usuário. |
| Reinício e recuperação manuais | Não exige ferramentas, mas a indisponibilidade dura até alguém perceber e agir. |
| Service mesh (Istio, Linkerd) para retries e disjuntores | Resolve as táticas de forma centralizada, mas exige Kubernetes e conhecimento operacional que a equipe não tem na primeira versão. |

## Justificativa

A disponibilidade não entrou no top 4 porque boa parte dela já resulta das ADR-003 e ADR-004: isolamento da captura, fila e devolução de créditos. Mas essas decisões não cobriam pontos importantes: o Gateway e a Identidade como pontos únicos de falha, o comportamento diante de provedores externos fora do ar e as reservas de crédito que ficam presas quando um worker morre.

As táticas escolhidas usam ferramentas que já estão na arquitetura (RabbitMQ, Docker, outbox) e não exigem infraestrutura nova. O foco é garantir que o usuário sempre consiga acessar as provas já registradas, mesmo quando a captura ou os provedores externos falham.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Falhas ficam contidas no serviço ou integração de origem | **Complexidade:** retries, disjuntores e filas de mensagens mortas precisam ser configurados e testados |
| Usuário logado continua usando o sistema se a Identidade cair | **Revogação mais lenta:** com validação local do JWT, um token revogado continua válido até expirar (reforça o prazo curto da ADR-001) |
| Nenhum evento perdido | **Duplicidade:** mensagens podem ser entregues mais de uma vez, então todos os consumidores precisam ser idempotentes |
| Recuperação automática da maioria das falhas | **Custo:** instâncias duplicadas de Gateway e Identidade e ferramentas de monitoramento |
| Créditos não ficam presos quando um worker morre | **Ajuste de parâmetros:** número de tentativas, timeouts e tempo de expiração precisam ser calibrados com testes |

## Consequências

**Positivas**
- O acesso às provas já registradas fica protegido das falhas da parte mais instável do sistema (captura e provedores externos).
- O comportamento em cada tipo de falha fica definido antes da implementação, o que facilita os testes.
- Fecha a lacuna das reservas de crédito presas, deixada em aberto pelas ADR-003 e ADR-004.

**Negativas / riscos**
- O banco de dados compartilhado e o broker continuam como pontos únicos de falha na primeira versão. A replicação deles exigirá uma nova ADR.
- Sem testes de falha (derrubar containers, simular provedor fora do ar), as táticas podem não funcionar quando forem necessárias.
