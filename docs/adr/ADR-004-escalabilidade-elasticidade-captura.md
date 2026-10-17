# ADR-004: Escalabilidade da captura como característica arquitetural prioritária

- **Status:** Aceito
- **Data:** 13/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Elaborado com apoio de IA em sala; revisado e aprovado pela equipe
- **Revisão 1 (17/10/2026):** Título alterado de "Escalabilidade e elasticidade" para "Escalabilidade da captura" (a escala automática fica como evolução), separação entre fila para capturas de tela e alocação de sessão para vídeo, isolamento de cada captura e metas de verificação. Revisão feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 2 (17/10/2026):** Correções de coerência com as demais ADRs: número de instâncias alinhado às ADR-005 e ADR-011 e custo ajustável à demanda (a escala é manual na primeira versão). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 3 (17/10/2026):** Checklist de revisão (A1: captura interrompida não é refeita; A6: pool de containers limpos de uso único; D1: cabeçalho). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 4 (17/10/2026):** Coerência com as demais ADRs: "workers que escalam sozinhos" trocado por "de forma independente", para não sugerir escala automática, que fica como evolução (item 4). Feita com apoio de IA fora de sala.

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

Adotar **Escalabilidade da captura** como característica arquitetural prioritária (4ª do top 4), implementada pelos seguintes mecanismos:

1. **Separar a API do Serviço de Captura dos workers.** A API, leve, recebe pedidos, valida a reserva de créditos e mostra o status. Os workers, pesados, executam o navegador.
2. **Fila de trabalhos** (RabbitMQ) entre a API e os workers para as **capturas de tela**, absorvendo picos sem rejeitar pedidos. O vídeo de navegação segue o item 7, porque o usuário precisa estar presente.
3. **Workers sem estado entre trabalhos e em containers**, com Playwright e Chromium. Cada worker tem um limite fixo de capturas simultâneas e de memória. O estado de uma sessão de vídeo existe só enquanto ela dura.
   - Cada captura roda num navegador novo, em container sem privilégios e numa rede isolada, descartado ao final (ADR-001, ADR-002). Isso protege a rede interna contra URLs maliciosas e garante um ambiente limpo para cada prova.
4. **Escala horizontal** pelo número de workers, aumentado ou reduzido manualmente com Docker Compose na primeira versão. A **elasticidade** (escala automática) fica como evolução, num orquestrador, usando como gatilho o tamanho da fila de tela e a ocupação do pool de vídeo.
5. **Envio direto ao object storage** (MinIO ou compatível com S3) a partir do worker, sem passar os arquivos pela API nem pelo banco.
6. **Limites operacionais:** timeout de carregamento de página, duração máxima do vídeo e tamanho máximo de arquivo, para que uma captura não monopolize um worker.
7. **Vídeo de navegação por alocação de sessão, não por fila.** Como o usuário navega ao vivo (ADR-002), a captura de vídeo reserva um navegador num pool de workers de vídeo, com limite de sessões simultâneas. O pool guarda containers limpos já iniciados; cada um é usado numa única sessão e descartado em seguida (item 3), o que também reduz o tempo de início. Sem navegador livre, o usuário vê a posição de espera ou é convidado a tentar depois, em vez de entrar numa fila longa. A sessão fica presa ao worker durante a gravação, e o worker só é desligado depois de concluir ou cancelar as sessões em andamento.
8. **Relatório e zip** (REQ14, REQ15) gerados por workers assíncronos do Serviço de Registros e Relatórios, que escalam do mesmo modo.

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Espera na fila de captura de tela | Até 2 minutos em 95% dos casos, no cenário de referência |
| Início de sessão de vídeo | Navegador alocado em até 10 segundos quando houver vaga no pool |
| Pico de captura | Login, listagem e download mantêm as metas da ADR-010 durante o pico |
| Inclusão de worker | Aumenta a vazão de capturas sem mudança de código ou de outro serviço |
| Capturas com navegador reaproveitado de outra captura | Zero |

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Captura síncrona dentro da requisição HTTP da API | Trava a API durante a captura, sujeita a timeouts e faz picos de captura derrubarem login, compra e listagem. |
| Escala vertical (um servidor maior) | Tem limite físico, é ponto único de falha e o custo não acompanha a demanda: paga-se pelo pico o tempo todo. |
| Funções serverless (AWS Lambda e similares) | Limite de tempo de execução e de memória, cold start do navegador e falta de suporte a sessões interativas longas tornam o vídeo inviável. |
| Escalar o sistema inteiro junto (monólito replicado) | Desperdiça recursos replicando partes leves para atender a parte pesada. |
| Fila única para tela e vídeo | Mais simples, mas o usuário do vídeo ficaria esperando na frente da tela sem previsão, e a sessão poderia começar quando ele já tivesse saído. |
| Priorizar disponibilidade em vez de escalabilidade no top 4 | A disponibilidade é atendida em boa parte pelo isolamento dos serviços e pela fila desta mesma decisão. A escalabilidade da captura é o requisito que mais influencia o desenho dos componentes. |

## Justificativa

A escalabilidade da captura entra no top 4 porque é o **principal fator de custo e o principal gargalo** do sistema. Ela também é a razão técnica mais forte para separar o sistema em serviços. Sem ela, um monólito modular bastaria.

Ao isolar a captura em workers que escalam de forma independente dos demais serviços, o restante do sistema fica protegido de picos de carga, o que também melhora a disponibilidade. E o custo de infraestrutura pode ser ajustado ao uso: manualmente na primeira versão e automaticamente quando a escala automática for adotada.

A arquitetura service-based permite essa decisão sem a complexidade dos microsserviços. Só o Serviço de Captura e os workers de relatório precisam escalar com a carga. Os outros serviços rodam com uma instância na primeira versão, exceto o Gateway e a Identidade, que têm duas por disponibilidade (ADR-005).

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Picos de captura não afetam login, compras e consultas | **Complexidade:** fila, workers, object storage e acompanhamento de status adicionam componentes e pontos de falha |
| Custo de infraestrutura ajustável à demanda (manualmente na primeira versão) | **Latência em pico:** com fila cheia, a captura entra em espera e o usuário aguarda para começar, o que exige mostrar a posição na fila ou o status |
| Capacidade de crescer para clientes empresariais (D1) adicionando workers | **Elasticidade limitada no vídeo:** sessões interativas são presas a um worker, que não pode ser desligado no meio, o que torna a redução de escala mais lenta e cara |
| Falha de um worker afeta só as capturas daquele worker | **Novos gargalos:** banco compartilhado, broker e object storage podem virar o limite quando houver muitos workers |
| Limites operacionais protegem contra sites lentos ou abusivos | **Usabilidade:** a duração máxima do vídeo e os timeouts restringem o que o usuário consegue registrar |
| | **Observabilidade obrigatória:** é preciso monitorar o tamanho da fila, o tempo de captura e o uso de recursos dos workers para decidir quando escalar |

## Consequências

**Positivas**
- O sistema absorve variações de demanda sem mudar a arquitetura.
- A fila também dá tolerância a falhas: a queda de um worker afeta só as capturas em andamento nele, que viram `FALHOU` com devolução do crédito (ADR-003, ADR-005). Capturas interrompidas não são refeitas automaticamente, porque uma nova tentativa seria outra captura, em outro momento.
- Relatório e zip seguem o mesmo padrão assíncrono, deixando o modelo de processamento uniforme.

**Negativas / riscos**
- A escala automática (elasticidade) fica fora da primeira versão e precisará de um orquestrador e de uma nova ADR.
- Criar um container ou navegador novo por captura aumenta o tempo de início de cada captura, o que precisa ser medido. No vídeo, o pool de containers pré-iniciados (item 7) reduz esse efeito.
- Os valores dos limites (duração do vídeo, timeouts, capturas simultâneas por worker) não estão nos requisitos e precisam ser definidos e validados.
- O uso de recursos do Chromium em produção precisa ser medido cedo, para dimensionar os workers e estimar o custo por registro.
