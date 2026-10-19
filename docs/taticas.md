# Táticas arquiteturais do Veridit

Táticas adotadas para cada uma das dez características arquiteturais do [README](../README.md), com o objetivo principal de cada uma. As táticas foram extraídas das [ADRs](adr/), sem alterá-las. A coluna **Item** indica o número do item na seção "Decisão" da ADR correspondente, onde está o detalhamento.

As características 1 a 4 formam o top 4 (ADR-001 a ADR-004).

## 1. Segurança ([ADR-001](adr/ADR-001-seguranca.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Autenticação centralizada com JWT de curta duração e token de renovação revogável | Ter um único ponto confiável de identificação do usuário | 1 |
| Validação do token no Gateway e em cada serviço | Impedir que o acesso à rede interna baste para chegar aos serviços (defesa em profundidade) | 2 |
| Rotas públicas explícitas, com webhooks protegidos por assinatura e consulta ao provedor | Garantir que nenhuma rota fique aberta por engano | 3 |
| Autorização por perfil e por dono do recurso | Impedir que um usuário acesse registros, relatórios ou arquivos de outro | 4 |
| Senhas com hash forte e lento (Argon2 ou bcrypt) | Proteger as senhas mesmo se o banco vazar | 5 |
| Recuperação de senha com token de uso único e prazo curto | Evitar o sequestro de contas pela recuperação de senha | 6 |
| Logout efetivo, com revogação do token de renovação | Encerrar a sessão de verdade no servidor (REQ04) | 7 |
| Confirmação do e-mail e validação do CPF no cadastro | Garantir que o e-mail de recuperação pertence ao usuário e que o CPF é válido | 8 |
| TLS em trânsito e criptografia em repouso | Proteger dados sensíveis e evidências contra interceptação e acesso ao disco | 9 |
| Schema de pagamentos isolado, com credenciais próprias | Reduzir a área de ataque sobre os dados financeiros | 10 |
| Limite de tentativas de login e de recuperação | Bloquear ataques de força bruta | 11 |
| Segredos fora do código | Evitar o vazamento de credenciais pelo repositório | 12 |
| Isolamento de rede dos workers de captura | Impedir que uma URL maliciosa alcance a rede interna (SSRF) | 13 |
| Downloads por link temporário assinado | Expor o storage à internet só para downloads autorizados e por poucos minutos | 14 |
| Autenticação entre serviços com credencial própria | Garantir que só serviços autorizados chamem as rotas internas | 15 |

## 2. Integridade e auditabilidade das evidências ([ADR-002](adr/ADR-002-integridade-auditabilidade-evidencias.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Captura feita no servidor, com navegador novo a cada captura | Garantir que o conteúdo não foi manipulado no dispositivo do usuário | 1 |
| Navegação remota no vídeo, transmitida pelo proxy do Gateway | Gravar no servidor mesmo quando o usuário navega ao vivo | 2 |
| Hash SHA-256 de cada arquivo logo após a captura | Permitir detectar qualquer alteração posterior | 3 |
| Manifesto do registro, com o próprio hash | Reunir num só documento verificável a origem, o momento e os arquivos da captura | 4 |
| Metadados técnicos no manifesto (IP, DNS, certificado TLS, cabeçalhos, HTML) | Demonstrar que o conteúdo veio de fato daquele site | 5 |
| Assinatura digital do manifesto, com chave fora dos workers e do banco | Impedir que alguém troque arquivos e hashes juntos sem ser detectado (não-repúdio) | 6 |
| Hash do manifesto no e-mail de conclusão | Criar um registro externo e datado, fora do controle do Veridit | 7 |
| Data e hora do servidor, sincronizadas por NTP e em UTC | Garantir um horário confiável, que não depende do relógio do cliente | 8 |
| Armazenamento imutável (versionamento e bloqueio de escrita) | Impedir que uma evidência gravada seja sobrescrita ou editada | 9 |
| Relatório e zip com manifesto, assinatura e hashes | Permitir que terceiros verifiquem a integridade sem depender do Veridit | 10 |
| Trilha de auditoria somente de inclusão | Registrar quem fez o quê e quando, sem permitir alteração | 11 |
| Máquina de estados do registro | Garantir que só registros concluídos gerem relatório e zip, evitando prova incompleta | 12 |

## 3. Confiabilidade transacional de pagamentos e créditos ([ADR-003](adr/ADR-003-confiabilidade-transacional-pagamentos.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Livro-razão de créditos somente de inclusão | Manter o saldo auditável e reconstruível a partir das movimentações | 1 |
| Máquina de estados do pagamento | Rejeitar transições inválidas de pagamento | 2 |
| Idempotência nos webhooks (id do pagamento ou txid como chave única) | Evitar crédito em dobro com notificações repetidas | 3 |
| Validação da origem do webhook e consulta ativa ao provedor | Evitar a aprovação de pagamentos com notificações falsas | 4 |
| Reconciliação periódica dos pagamentos pendentes | Recuperar webhooks perdidos e marcar Pix expirados | 5 |
| Reserva → consumo ou devolução, idempotentes pelo id do registro | Garantir que nenhuma captura rode sem saldo e que o crédito volte se ela falhar | 6 |
| Outbox transacional | Publicar eventos sem perdê-los nem duplicá-los | 7 |
| Preço congelado na compra | Impedir que mudanças nos pacotes (D2) alterem compras já feitas | 8 |
| Débito dos créditos em estorno pelo provedor | Manter o saldo correto quando um pagamento é desfeito ou contestado | 9 |

## 4. Escalabilidade da captura ([ADR-004](adr/ADR-004-escalabilidade-elasticidade-captura.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Separar a API de Captura dos workers | Escalar só a parte pesada (navegador), mantendo a API leve | 1 |
| Fila de trabalhos para a captura de tela | Absorver picos de demanda sem rejeitar pedidos | 2 |
| Workers sem estado, em containers descartáveis | Permitir adicionar ou remover workers livremente, com ambiente limpo a cada prova | 3 |
| Escala horizontal pelo número de workers | Aumentar a vazão de capturas sem mudar código nem outros serviços | 4 |
| Envio direto ao object storage a partir do worker | Tirar o tráfego de arquivos da API e do banco | 5 |
| Limites operacionais (timeout, duração e tamanho máximos) | Impedir que uma captura monopolize um worker | 6 |
| Vídeo por alocação de sessão num pool | Atender a navegação ao vivo sem deixar o usuário numa fila longa | 7 |
| Workers assíncronos para relatório e zip | Escalar a geração de relatórios e zips da mesma forma que a captura | 8 |

## 5. Disponibilidade e tolerância a falhas ([ADR-005](adr/ADR-005-disponibilidade-tolerancia-falhas.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Isolamento por serviço (bulkhead), com limites de CPU e memória | Impedir que uma sobrecarga na captura afete os demais serviços | 1 |
| Validação local do token no Gateway | Manter logado quem já entrou, mesmo com a Identidade fora do ar | 2 |
| Gateway e Identidade com pelo menos duas instâncias | Eliminar pontos únicos de falha na entrada do sistema | 3 |
| Mensageria durável, com confirmação manual | Não perder trabalhos quando um worker cai | 4 |
| Novas tentativas com espera crescente e fila de mensagens mortas | Superar falhas temporárias e isolar as permanentes para análise | 5 |
| Timeouts e disjuntor em toda chamada síncrona | Falhar rápido em vez de prender conexões com um serviço fora do ar | 6 |
| Degradação controlada | Definir o que continua funcionando quando cada dependência falha | 7 |
| Expiração de reservas presas | Devolver créditos de capturas cujo worker morreu sem avisar | 8 |
| Verificação de saúde e reinício automático | Recuperar containers com falha sem intervenção manual | 9 |
| Monitoramento e alertas | Detectar falhas e filas acumuladas antes que afetem o usuário | 10 |

## 6. Durabilidade e recuperação das evidências ([ADR-006](adr/ADR-006-durabilidade-recuperacao-evidencias.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Registro só concluído depois do armazenamento confirmado | Garantir que nenhum registro `CONCLUIDO` fique sem os arquivos | 1 |
| Redundância no object storage (*erasure coding*) | Tolerar a perda de discos sem perder dados | 2 |
| Cópia das evidências em outro local | Sobreviver à perda do ambiente principal | 3 |
| Proteção contra exclusão na cópia, com credenciais separadas | Impedir que um invasor ou um erro no ambiente principal apague a cópia | 4 |
| Backup contínuo do banco (WAL e backup diário) | Restaurar os metadados a qualquer momento recente (RPO de 5 minutos) | 5 |
| Manifesto guardado junto dos arquivos | Reconstruir os dados essenciais das evidências mesmo se o banco for perdido | 6 |
| Verificação periódica de integridade | Detectar corrupção silenciosa e restaurar a partir da cópia | 7 |
| Testes de restauração mensais | Garantir que os backups funcionam de fato | 8 |

## 7. Conformidade legal e privacidade — LGPD ([ADR-007](adr/ADR-007-conformidade-lgpd.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Minimização dos dados coletados | Coletar só o necessário para cada finalidade | 1 |
| Base legal por finalidade, num inventário de tratamento | Justificar legalmente cada tratamento de dados | 2 |
| Segregação e criptografia de CPF e telefone (com hash HMAC do CPF para busca) | Proteger dados sensíveis sem impedir a checagem de cadastro duplicado | 3 |
| Logs sem dados pessoais | Evitar o vazamento de dados pessoais por logs e mensagens de erro | 4 |
| Política de retenção por tipo de dado | Guardar cada dado só pelo tempo necessário | 5 |
| Expurgo controlado das evidências, com aprovação dupla | Conciliar a imutabilidade das provas com o direito de exclusão | 6 |
| Direitos do titular na plataforma | Permitir acesso, correção e encerramento da conta pelo próprio usuário | 7 |
| Canal para pedidos de terceiros | Atender pessoas que aparecem nas evidências, avaliando caso a caso | 8 |
| Governança (encarregado de dados, política de privacidade e resposta a incidentes) | Cumprir as obrigações organizacionais da LGPD | 9 |

## 8. Interoperabilidade ([ADR-008](adr/ADR-008-interoperabilidade.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Portas e adaptadores para cada provedor externo | Trocar ou adicionar provedores sem mexer na lógica de negócio | 1 |
| Camada anticorrupção para webhooks | Isolar o sistema dos formatos de cada provedor de pagamento | 2 |
| Um único provedor de Pix na primeira versão (Mercado Pago) | Ter um só contrato, webhook e reconciliação | 3 |
| Protocolos e padrões abertos (S3, SMTP, HTTP/JSON, JWT) | Evitar dependência de um fornecedor específico | 4 |
| Contratos das APIs internas em OpenAPI, versionados | Permitir evoluir as APIs sem quebrar os consumidores | 5 |
| Eventos com esquema versionado e catálogo | Garantir compatibilidade e idempotência na troca de mensagens | 6 |
| Configuração externa | Trocar credenciais, URLs e adaptadores sem alterar o código | 7 |
| Testes de contrato em sandbox e entre serviços | Detectar quebras de integração antes da implantação | 8 |

## 9. Extensibilidade e manutenibilidade ([ADR-009](adr/ADR-009-extensibilidade-manutenibilidade.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Tipos de usuário como perfis (D1) | Incluir empresas sem mudar a autenticação nem os outros perfis | 1 |
| Pacotes de créditos como dados, com área administrativa (D2) | Criar pacotes sem nova versão do sistema | 2 |
| Tipos de captura e formatos de saída como estratégias | Incluir novos tipos sem alterar o fluxo principal | 3 |
| Mesma estrutura em camadas em todos os serviços | Facilitar o entendimento e isolar o domínio de frameworks e infraestrutura | 4 |
| Cada serviço só acessa o próprio schema | Evitar o acoplamento entre serviços pelo banco compartilhado | 5 |
| Repositório único, com biblioteca compartilhada mínima | Compartilhar só contratos e utilitários, nunca regras de negócio | 6 |
| Qualidade automatizada na integração contínua | Detectar regressões cedo, com a equipe em rotatividade | 7 |
| Decisões registradas em ADR | Preservar o porquê de cada mudança arquitetural | 8 |

## 10. Desempenho ([ADR-010](adr/ADR-010-desempenho.md))

| Tática | Objetivo principal | Item |
|---|---|---|
| Separar operações rápidas e lentas (assíncronas) | Manter a interface responsiva durante captura, relatório e zip | 1 |
| Relatório e zip gerados uma vez e reaproveitados | Evitar gerar de novo o mesmo arquivo a cada download | 2 |
| Download direto do object storage | Tirar o tráfego de arquivos da API e do Gateway | 3 |
| Listagem paginada e indexada | Responder a listagem e os detalhes em até 1 s | 4 |
| Token validado no Gateway sem chamada à Identidade | Eliminar uma chamada extra em cada requisição | 5 |
| Hash de senha calibrado (250 a 500 ms) | Equilibrar a segurança da senha e o tempo de login | 6 |
| Pool e limite de conexões por serviço | Impedir que um serviço esgote o banco dos outros | 7 |
| Interface web estática, com compressão e cache | Abrir a página rapidamente | 8 |
| Acompanhamento de status leve | Mostrar o progresso sem sobrecarregar a API | 9 |
| Testes de carga antes de cada entrega | Verificar as metas de tempo de resposta | 10 |

## Táticas do estilo arquitetural

O estilo service-based ([ADR-011](adr/ADR-011-estilo-arquitetural-service-based.md)) também traz táticas que atendem a várias características ao mesmo tempo:

| Tática | Objetivo principal | Características |
|---|---|---|
| Serviços de domínio de granularidade grossa | Isolar a captura e os pagamentos sem o custo dos microsserviços | 1, 3, 4, 5 |
| Um banco PostgreSQL com um schema por serviço | Manter transações ACID locais sem sagas | 3, 9 |
| Comunicação síncrona HTTP/JSON para operações rápidas | Simplificar consultas e comandos curtos | 10 |
| Comunicação assíncrona por fila para operações demoradas | Desacoplar serviços e absorver picos | 4, 5, 10 |
| Object storage para evidências, relatórios e zips | Guardar arquivos grandes fora do banco, com imutabilidade | 2, 6 |
| Escala por serviço | Replicar só o que precisa escalar | 4, 5 |
| Execução com Docker Compose | Rodar o sistema completo localmente, dentro do prazo e do orçamento | Viabilidade |
