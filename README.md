# ESA_Trabalho

Projeto arquitetural do sistema **Veridit**, desenvolvido para a disciplina de Engenharia de Software I (Prof. Dr. Eduardo Almeida).

O Veridit é uma plataforma de registro de evidências digitais: o usuário (advogado ou usuário comum) compra créditos, captura conteúdo de sites da internet (tela ou vídeo de navegação) e obtém um relatório e um arquivo zip com os arquivos capturados, para uso como prova.

- Documento de requisitos: [`veridit-req.pdf`](veridit-req.pdf)
- Enunciado do trabalho: [`trabalhoII.pdf`](trabalhoII.pdf)
- Registros de decisões arquiteturais: [`docs/adr/`](docs/adr/)
- Componentes candidatos: [`docs/componentes.md`](docs/componentes.md)
- Tecnologias: [`docs/tecnologias.md`](docs/tecnologias.md)

---

## Arquitetura escolhida: Service-based

O sistema é dividido em poucos serviços de domínio, de granularidade grossa, implantados de forma independente. Os serviços compartilham uma instância de banco de dados, com schemas separados por domínio (o de pagamentos fica isolado). As operações longas usam processamento assíncrono por fila.

| Serviço | Responsabilidade | Requisitos |
|---|---|---|
| **Interface Web + API Gateway** | SPA do usuário, roteamento das requisições e validação do token de acesso | Todos |
| **Identidade** | Cadastro de advogados e usuários comuns, login, logout e recuperação de senha | REQ01–04, D1 |
| **Créditos e Pagamentos** | Pacotes de créditos, dados de faturamento, Mercado Pago, Pix (QR Code/chave), webhooks e registro de movimentação de créditos | REQ05–06, D2 |
| **Captura** | Sessão de captura, fila de trabalhos e workers com navegador sem interface que geram imagens e vídeos com hash e data/hora | REQ08, REQ09, REQ11 |
| **Registros e Relatórios** | Listagem e detalhes dos registros, geração do relatório e do arquivo zip | REQ12–15 |
| **Notificação** | Recebe eventos (pagamento aprovado, captura concluída, confirmação de e-mail do cadastro e recuperação de senha) e envia os e-mails | REQ07, REQ10, REQ11 |

### Por que Service-based é a mais apropriada para o Veridit

**1. Isola a parte pesada do sistema.** A captura de tela e de vídeo (REQ08–11) roda navegadores no servidor e consome muito CPU, memória e disco. Com um serviço de captura separado (fila para capturas de tela e pool de navegadores para vídeo), é possível aumentar ou reduzir os workers conforme a demanda, sem escalar o sistema inteiro. Se a captura falhar ou ficar lenta, login, compra, listagem e download continuam funcionando.

**2. Protege pagamentos e dados sensíveis.** Pagamentos e dados de faturamento (REQ05–07) ficam num serviço com schema e credenciais de banco isolados, e os dados de cadastro (CPF, OAB) ficam no Serviço de Identidade, criptografados (ADR-001, ADR-007). Isso reduz a área de ataque e facilita aplicar controles de segurança e da LGPD. Segurança é o atributo de qualidade explícito no documento de requisitos.

**3. Mantém a consistência onde ela importa.** Cada serviço trata seu domínio com transações ACID no próprio schema do banco compartilhado. Isso garante a integridade do saldo de créditos e dos metadados das evidências, sem a complexidade da consistência eventual, de sagas e de um banco por serviço que os microsserviços exigiriam.

**4. Combina com o assíncrono que já existe nos requisitos.** Confirmação de pagamento via webhook e Pix, captura demorada, PDF, zip e e-mails (REQ07, 10, 11) já são processos assíncronos por natureza. Uma fila entre os serviços de Captura, Registros e Notificação atende esses fluxos sem exigir que tudo seja orientado a eventos.

**5. Facilita a evolução prevista.** As decisões D1 (cadastro de empresas) e D2 (novos pacotes) indicam crescimento. Com fronteiras por domínio, as mudanças ficam no serviço afetado. Se no futuro algum serviço precisar de mais independência, ele pode virar um microsserviço sem reescrever o resto.

**6. Cabe na equipe e no prazo.** Com 6 serviços, banco compartilhado e ferramentas open source (PostgreSQL, RabbitMQ, MinIO, Playwright), o sistema pode ser executado localmente com Docker Compose. Os ganhos de modularidade e isolamento vêm com uma fração do custo operacional dos microsserviços. Isso é decisivo para uma equipe acadêmica com prazo curto.

### Alternativas consideradas

- **Monólito em camadas e monólito modular:** mais simples e baratos, mas escalam como um bloco só, e uma falha na captura ou um deploy com erro afeta todo o sistema, inclusive pagamentos. O monólito modular fica como plano B caso o prazo de implementação aperte.
- **Microsserviços:** máxima independência, mas exigem um banco por serviço e consistência eventual no saldo de créditos, com custo operacional desproporcional ao escopo de 15 requisitos e ao tamanho da equipe.
- **Orientada a eventos pura:** desacopla e escala bem, mas torna os fluxos difíceis de acompanhar e também exige consistência eventual nos créditos.
- **Serverless:** o limite de tempo de execução inviabiliza a captura de vídeo.

> A justificativa formal desta decisão, com a comparação entre os estilos considerados, está na [ADR-011](docs/adr/ADR-011-estilo-arquitetural-service-based.md).

---

## Top 10 características arquiteturais

**1. Segurança**
Único atributo de qualidade explícito nos requisitos ("acesso apenas por usuários cadastrados"). O sistema guarda dados pessoais sensíveis (CPF, OAB, endereço, telefone), processa pagamentos e armazena provas que não podem ser acessadas por terceiros. Exige autenticação (REQ03–04), recuperação de senha segura (REQ02), autorização por dono do registro (REQ12–15) e proteção dos dados armazenados e em trânsito. Aqui, segurança trata de **quem pode acessar** o quê. A garantia de que a prova não foi alterada é tratada na característica 2.

**2. Integridade e auditabilidade das evidências**
O valor do produto está em o conteúdo capturado servir de prova, então ele precisa ser verificável e à prova de adulteração. Embora a integridade faça parte de segurança em sentido amplo, aqui ela é tratada à parte porque o foco é outro: **provar a terceiros** (perito, juiz, parte contrária) que a evidência não mudou desde a captura (não-repúdio). Os campos do Apêndice A (id gerado pelo sistema, data/hora de início e fim, URL, nome e tamanho do arquivo, usuário responsável) mostram que o relatório (REQ14) e o zip (REQ15) precisam provar origem, momento e completude da captura. Isso pede hash dos arquivos, data/hora confiável, armazenamento imutável e trilha de auditoria.

**3. Confiabilidade transacional (pagamentos e créditos)**
A compra (REQ05–07) e o consumo de créditos envolvem dinheiro e integração externa assíncrona (webhooks do Mercado Pago e confirmação do Pix). Um webhook duplicado, perdido ou atrasado não pode gerar crédito em dobro, crédito perdido nem captura sem saldo. Exige idempotência, consistência no saldo e devolução do crédito quando a captura falhar. Esta característica trata de o sistema **estar correto**. Estar no ar é tratado na característica 5.

**4. Escalabilidade da captura**
A captura de tela e de vídeo (REQ08–11) usa navegadores sem interface no servidor, com alto consumo de recursos e carga variável. O serviço de captura e seus workers precisam escalar horizontalmente de forma independente, sem levar junto os serviços de identidade, pagamento ou consulta. Na primeira versão a escala é manual; a elasticidade (escala automática conforme o tamanho da fila) fica como evolução.

**5. Disponibilidade e tolerância a falhas**
Uma falha num worker de captura, no provedor de e-mail ou no gateway de pagamento não pode derrubar o sistema. O usuário deve continuar logando, listando registros e baixando relatórios. Serviços separados, filas e novas tentativas automáticas mantêm as falhas isoladas. Uma captura interrompida deve ficar com status claro (Apêndice A: "status"), nunca como prova incompleta.

**6. Durabilidade e recuperação das evidências**
A integridade garante que a prova não foi alterada, mas não que ela não se perca. Um disco corrompido, um bucket apagado por engano ou uma falha no banco podem destruir evidências que já estão sendo usadas em processos judiciais, e uma captura não pode ser refeita depois, porque o site muda. Exige backup do banco e do object storage, replicação dos arquivos de evidência, testes periódicos de restauração e metas definidas de perda máxima de dados e de tempo de recuperação.

**7. Conformidade legal e privacidade (LGPD)**
O sistema trata CPF, número da OAB, endereço e dados de faturamento, e produz material com finalidade jurídica. Exige pedir só os dados necessários, controlar o acesso, definir prazos de retenção e permitir exclusão ou anonimização. Isso influencia o modelo de dados e o isolamento do schema de pagamentos.

**8. Interoperabilidade**
O sistema depende de serviços externos: Mercado Pago, Pix (REQ06), provedor de e-mail (REQ07, 10, 11) e acesso a sites de terceiros para captura (REQ08). As integrações ficam atrás de adaptadores, permitindo trocar ou adicionar provedores sem afetar as regras de negócio.

**9. Extensibilidade e manutenibilidade**
O próprio documento de requisitos já anuncia mudanças: D1 (empresas como novo tipo de usuário) e D2 (novos pacotes de créditos). Os tipos de conteúdo capturado (tela, vídeo) e os formatos de saída (relatório, zip) também tendem a aumentar. Além disso, o enunciado da disciplina prevê etapas de implementação e evolução depois desta fase arquitetural, feitas por uma equipe pequena e com rotatividade de tarefas. O sistema precisa aceitar essas mudanças com alterações localizadas, fáceis de entender e de testar.

**10. Desempenho**
Não há requisito explícito de tempo de resposta no documento, mas a característica é implícita na experiência de uso. Login, listagem (REQ12) e detalhes (REQ13) precisam responder rapidamente (meta inicial: menos de 1 s). Captura, geração de relatório (REQ14) e de zip (REQ15) são demoradas e devem rodar de forma assíncrona, com aviso por e-mail (REQ10, 11) ou acompanhamento de status, para que a interface não trave.

### Restrição transversal: viabilidade (custo e prazo)

Além das dez características, a arquitetura está sujeita a uma restrição de viabilidade: equipe acadêmica pequena, prazo curto e orçamento limitado de infraestrutura. Ela não é uma qualidade que o sistema entrega ao usuário, mas pesa em todas as decisões. Foi decisiva na escolha do estilo service-based em vez de microsserviços e na preferência por ferramentas open source executáveis localmente (ver ADR-004).
