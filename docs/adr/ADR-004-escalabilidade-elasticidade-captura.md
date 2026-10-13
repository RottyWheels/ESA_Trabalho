# ADR-004: Escalabilidade e elasticidade da captura como característica arquitetural prioritária

- **Status:** Proposto
- **Data:** 13/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Rascunho elaborado com apoio de IA em sala, sujeito à revisão e aprovação da equipe

## Contexto

A captura de conteúdo (REQ08 a REQ11) é a operação **mais cara** do Veridit. Cada registro abre um navegador sem interface (Chromium) no servidor, carrega um site de terceiros e produz capturas de tela ou um **vídeo de navegação**. Cada sessão consome centenas de MB de memória, CPU para renderizar e codificar o vídeo, e disco e rede para enviar os arquivos.

A demanda tende a ser **irregular**: picos em horário comercial, prazos processuais e clientes empresariais no futuro (D1). As demais operações (login, compra, listagem) são leves e têm carga mais estável.

Como a captura precisa ser feita no servidor para garantir a integridade da evidência (ADR-002), não dá para transferir esse custo para o dispositivo do usuário.

## Requisitos e restrições relacionados

- **REQ08:** Iniciar registro de conteúdo (tela ou vídeo de navegação)
- **REQ09:** Concluir gravação de conteúdo
- **REQ10:** Confirmar gravação por e-mail, o que indica processamento assíncrono
- **REQ11:** Concluir captura de tela e confirmar por e-mail
- **REQ14 e REQ15:** Gerar relatório e zip, também operações pesadas
- **D1:** Cadastro futuro de empresas, com aumento esperado de volume
- **ADR-002:** A captura precisa acontecer no servidor
- **Restrição do projeto:** Equipe pequena e orçamento limitado de infraestrutura

## Decisão

Adotar **Escalabilidade e elasticidade da captura** como característica arquitetural prioritária (4ª do top 4), implementada pelos seguintes mecanismos:

1. **Separar a API do Serviço de Captura dos workers.** A API, leve, recebe pedidos, valida a reserva de créditos e mostra o status. Os workers, pesados, executam o navegador.
2. **Fila de trabalhos** (RabbitMQ) entre a API e os workers, absorvendo picos sem rejeitar pedidos.
3. **Workers sem estado entre trabalhos e em containers**, com Playwright e Chromium. Cada worker tem um limite fixo de capturas simultâneas e de memória.
4. **Escala horizontal** pelo número de workers, aumentado ou reduzido conforme o tamanho da fila (manualmente com Docker Compose na primeira versão, automaticamente em um orquestrador numa evolução).
5. **Envio direto ao object storage** (MinIO ou compatível com S3) a partir do worker, sem passar os arquivos pela API nem pelo banco.
6. **Limites operacionais:** timeout de carregamento de página, duração máxima do vídeo e tamanho máximo de arquivo, para que uma captura não monopolize um worker.
7. **Vídeo de navegação** em sessão interativa presa a um worker durante toda a gravação. O worker só é desligado depois de concluir ou cancelar as sessões em andamento.
8. **Relatório e zip** (REQ14, REQ15) gerados por workers assíncronos do Serviço de Registros e Relatórios, que escalam do mesmo modo.

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Captura síncrona dentro da requisição HTTP da API | Trava a API durante a captura, sujeita a timeouts e faz picos de captura derrubarem login, compra e listagem. |
| Escala vertical (um servidor maior) | Tem limite físico, é ponto único de falha e o custo não acompanha a demanda: paga-se pelo pico o tempo todo. |
| Funções serverless (AWS Lambda e similares) | Limite de tempo de execução e de memória, cold start do navegador e falta de suporte a sessões interativas longas tornam o vídeo inviável. |
| Escalar o sistema inteiro junto (monólito replicado) | Desperdiça recursos replicando partes leves para atender a parte pesada. |
| Priorizar disponibilidade em vez de escalabilidade no top 4 | A disponibilidade é atendida em boa parte pelo isolamento dos serviços e pela fila desta mesma decisão. A escalabilidade da captura é o requisito que mais influencia o desenho dos componentes. |

## Justificativa

A escalabilidade da captura entra no top 4 porque é o **principal fator de custo e o principal gargalo** do sistema. Ela também é a razão técnica mais forte para separar o sistema em serviços. Sem ela, um monólito modular bastaria.

Ao isolar a captura em workers que escalam sozinhos, o restante do sistema fica protegido de picos de carga, o que também melhora a disponibilidade. E o custo de infraestrutura passa a acompanhar o uso real.

A arquitetura service-based permite essa decisão sem a complexidade dos microsserviços. Só o Serviço de Captura e os workers de relatório precisam escalar. Os outros serviços podem rodar com uma única instância na primeira versão.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Picos de captura não afetam login, compras e consultas | **Complexidade:** fila, workers, object storage e acompanhamento de status adicionam componentes e pontos de falha |
| Custo de infraestrutura proporcional à demanda | **Latência em pico:** com fila cheia, a captura entra em espera e o usuário aguarda para começar, o que exige mostrar a posição na fila ou o status |
| Capacidade de crescer para clientes empresariais (D1) adicionando workers | **Elasticidade limitada no vídeo:** sessões interativas são presas a um worker, que não pode ser desligado no meio, o que torna a redução de escala mais lenta e cara |
| Falha de um worker afeta só as capturas daquele worker | **Novos gargalos:** banco compartilhado, broker e object storage podem virar o limite quando houver muitos workers |
| Limites operacionais protegem contra sites lentos ou abusivos | **Usabilidade:** a duração máxima do vídeo e os timeouts restringem o que o usuário consegue registrar |
| | **Observabilidade obrigatória:** é preciso monitorar o tamanho da fila, o tempo de captura e o uso de recursos dos workers para decidir quando escalar |

## Consequências

**Positivas**
- O sistema absorve variações de demanda sem mudar a arquitetura.
- A fila também dá tolerância a falhas: trabalhos de um worker que caiu podem ser reenfileirados (com estorno de crédito se a captura não puder ser retomada, conforme ADR-003).
- Relatório e zip seguem o mesmo padrão assíncrono, deixando o modelo de processamento uniforme.

**Negativas / riscos**
- A escala automática fica fora da primeira versão e precisará de um orquestrador e de uma nova ADR.
- Os valores dos limites (duração do vídeo, timeouts, capturas simultâneas por worker) não estão nos requisitos e precisam ser definidos e validados.
- O uso de recursos do Chromium em produção precisa ser medido cedo, para dimensionar os workers e estimar o custo por registro.
