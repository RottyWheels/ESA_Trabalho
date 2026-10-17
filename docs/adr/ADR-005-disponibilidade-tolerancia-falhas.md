# ADR-005: Disponibilidade e tolerância a falhas

- **Status:** Aceito
- **Data:** 17/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Rascunho elaborado com apoio de IA fora de sala, com autorização excepcional do professor, sujeito à revisão e aprovação da equipe

## Contexto

O Veridit depende de vários componentes que podem falhar de forma independente: os workers de captura (navegador no servidor), o provedor de e-mail, o Mercado Pago e o provedor de Pix, o broker de mensagens, o banco de dados e os próprios sites de terceiros que o usuário quer registrar.

Para o usuário, as funções mais críticas no dia a dia são entrar no sistema, consultar os registros e baixar relatórios e zips, porque é isso que ele leva a um processo. Uma falha na captura ou no e-mail não pode impedir o acesso às provas já registradas.

A ADR-001 também apontou o API Gateway e o Serviço de Identidade como pontos críticos de disponibilidade, e as ADR-003 e ADR-004 já introduziram fila, estorno e reprocessamento. Esta ADR reúne essas táticas e completa o que falta.

## Requisitos e restrições relacionados

- **REQ03 e REQ04:** Logar e sair do sistema
- **REQ07, REQ10 e REQ11:** E-mails de confirmação, que dependem de provedor externo
- **REQ08 a REQ11:** Captura, que depende de workers e de sites de terceiros
- **REQ12 a REQ15:** Listar, detalhar, gerar relatório e zip, as funções de consulta mais críticas
- **Apêndice A:** Campo "status" dos registros realizados
- **ADR-001:** Gateway e Identidade como pontos críticos
- **ADR-003 e ADR-004:** Fila, reserva e estorno de créditos
- **Restrição do projeto:** Orçamento limitado, sem infraestrutura em várias regiões

## Decisão

Tratar **disponibilidade e tolerância a falhas** com as seguintes táticas:

1. **Isolamento por serviço (bulkhead).** Cada serviço roda em containers próprios, com limites de CPU e memória. Uma falha ou sobrecarga na captura não consome recursos dos demais.
2. **Validação local do token no Gateway.** O Gateway valida o JWT com a chave pública do Serviço de Identidade, sem chamá-lo a cada requisição. Se a Identidade cair, quem já está logado continua usando o sistema; só o login e a renovação ficam indisponíveis.
3. **Gateway e Identidade com pelo menos duas instâncias.** Os dois não guardam estado de sessão em memória, então podem ser replicados atrás de um balanceador.
4. **Mensageria durável.** Filas e mensagens persistentes no RabbitMQ, com confirmação manual (*ack*) só depois do processamento. Se um worker cai no meio, a mensagem volta para a fila.
5. **Novas tentativas com espera crescente e fila de mensagens mortas.** Falhas temporárias (e-mail, provedor de pagamento, site lento) são repetidas com espera exponencial e variação aleatória. Depois do limite de tentativas, a mensagem vai para uma fila de mensagens mortas, que gera alerta para análise.
6. **Timeouts e disjuntor (*circuit breaker*)** em toda chamada síncrona: Gateway para os serviços, Captura para Créditos, e adaptadores dos provedores externos. Um provedor fora do ar falha rápido em vez de prender conexões.
7. **Degradação controlada.** Cada falha tem um comportamento definido:
   - Créditos fora do ar: novas capturas são recusadas com mensagem clara; consulta e download continuam.
   - Provedor de e-mail fora do ar: o evento fica na outbox (ADR-003) e o e-mail é enviado quando ele voltar; o status também aparece na tela.
   - Mercado Pago fora do ar: oferecer Pix, e vice-versa.
8. **Expiração de reservas presas.** Uma tarefa periódica desfaz reservas de crédito de registros que ficaram em andamento além do tempo máximo de captura (ADR-004), marca o registro como `FALHOU` e devolve o crédito (ADR-003).
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

A disponibilidade não entrou no top 4 porque boa parte dela já resulta das ADR-003 e ADR-004: isolamento da captura, fila e estorno. Mas essas decisões não cobriam pontos importantes: o Gateway e a Identidade como pontos únicos de falha, o comportamento diante de provedores externos fora do ar e as reservas de crédito que ficam presas quando um worker morre.

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
