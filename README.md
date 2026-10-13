# ESA_Trabalho

Projeto arquitetural do sistema **Veridit**, desenvolvido para a disciplina de Engenharia de Software I (Prof. Dr. Eduardo Almeida).

O Veridit é uma plataforma de registro de evidências digitais: o usuário (advogado ou usuário comum) compra créditos, captura conteúdo de sites da internet (tela ou vídeo de navegação) e obtém um relatório e um arquivo zip com os arquivos capturados, para uso como prova.

- Documento de requisitos: [`veridit-req.pdf`](veridit-req.pdf)
- Enunciado do trabalho: [`trabalhoII.pdf`](trabalhoII.pdf)
- Registros de decisões arquiteturais: [`docs/adr/`](docs/adr/)

---

## Arquitetura escolhida: Service-based

O sistema é dividido em poucos serviços de domínio, de granularidade grossa, implantados de forma independente. Os serviços compartilham uma instância de banco de dados, com schemas separados por domínio (o de pagamentos fica isolado). As operações longas usam processamento assíncrono por fila.

| Serviço | Responsabilidade | Requisitos |
|---|---|---|
| **Interface Web + API Gateway** | SPA do usuário, roteamento das requisições e validação do token de acesso | Todos |
| **Identidade** | Cadastro de advogados e usuários comuns, login, logout e recuperação de senha | REQ01–04, D1 |
| **Créditos e Pagamentos** | Pacotes de créditos, dados de faturamento, Mercado Pago, Pix (QR Code/chave), webhooks e registro de movimentação de créditos | REQ05–06, D2 |
| **Captura** | Sessão de captura, fila de trabalhos e workers com navegador sem interface que geram imagens e vídeos com hash e data/hora | REQ08–11 |
| **Registros e Relatórios** | Listagem e detalhes dos registros, geração do relatório e do arquivo zip | REQ12–15 |
| **Notificação** | Recebe eventos (pagamento confirmado, captura concluída) e envia os e-mails | REQ07, REQ10, REQ11 |

### Por que Service-based é a mais apropriada para o Veridit

**1. Isola a parte pesada do sistema.** A captura de tela e de vídeo (REQ08–11) roda navegadores no servidor e consome muito CPU, memória e disco. Com um serviço de captura separado, alimentado por fila, é possível aumentar ou reduzir os workers conforme a demanda, sem escalar o sistema inteiro. Se a captura falhar ou ficar lenta, login, compra, listagem e download continuam funcionando.

**2. Protege pagamentos e dados sensíveis.** Pagamentos (REQ05–07) e dados pessoais (CPF, OAB, endereço) ficam num serviço com banco isolado. Isso reduz a área de ataque e facilita aplicar controles de segurança e da LGPD. Segurança é o atributo de qualidade explícito no documento de requisitos.

**3. Mantém a consistência onde ela importa.** Cada serviço trata seu domínio com transações ACID no próprio banco. Isso garante a integridade do saldo de créditos e dos metadados das evidências, sem a complexidade da consistência eventual, de sagas e de um banco por serviço que os microsserviços exigiriam.

**4. Combina com o assíncrono que já existe nos requisitos.** Confirmação de pagamento via webhook e Pix, captura demorada, PDF, zip e e-mails (REQ07, 10, 11) já são processos assíncronos por natureza. Uma fila entre os serviços de Captura, Registros e Notificação atende esses fluxos sem exigir que tudo seja orientado a eventos.

**5. Facilita a evolução prevista.** As decisões D1 (cadastro de empresas) e D2 (novos pacotes) indicam crescimento. Com fronteiras por domínio, as mudanças ficam no serviço afetado. Se no futuro algum serviço precisar de mais independência, ele pode virar um microsserviço sem reescrever o resto.

**6. Cabe na equipe e no prazo.** Com 4 a 6 serviços, banco compartilhado e ferramentas open source (PostgreSQL, RabbitMQ, MinIO, Playwright), o sistema pode ser executado localmente com Docker Compose. Os ganhos de modularidade e isolamento vêm com uma fração do custo operacional dos microsserviços. Isso é decisivo para uma equipe acadêmica com prazo curto.

### Alternativas consideradas

- **Monólito modular:** mais simples e barato, mas escala como um bloco só, e uma falha na captura ou um deploy com erro afeta todo o sistema, inclusive pagamentos. Fica como plano B caso o prazo de implementação aperte.
- **Microsserviços orientados a eventos:** máxima escalabilidade e independência, mas a complexidade operacional (orquestração, rastreamento distribuído, versionamento de eventos) e a consistência eventual em créditos e evidências são desproporcionais ao escopo de 15 requisitos e ao tamanho da equipe.

> A justificativa formal desta decisão será registrada como ADR em [`docs/adr/`](docs/adr/), durante as sessões em sala, conforme as regras da disciplina.

---

## Top 10 características arquiteturais

**1. Segurança**
Único atributo de qualidade explícito nos requisitos ("acesso apenas por usuários cadastrados"). O sistema guarda dados pessoais sensíveis (CPF, OAB, endereço, telefone), processa pagamentos e armazena provas que não podem ser acessadas nem alteradas por terceiros. Exige autenticação (REQ03–04), recuperação de senha segura (REQ02), autorização por dono do registro (REQ12–15) e proteção dos dados armazenados e em trânsito.

**2. Integridade e auditabilidade das evidências**
O valor do produto está em o conteúdo capturado servir de prova, então ele precisa ser verificável e à prova de adulteração. Os campos do Apêndice A (id gerado pelo sistema, data/hora de início e fim, URL, nome e tamanho do arquivo, usuário responsável) mostram que o relatório (REQ14) e o zip (REQ15) precisam provar origem, momento e completude da captura. Isso pede hash dos arquivos, data/hora confiável, armazenamento imutável e trilha de auditoria.

**3. Confiabilidade transacional (pagamentos e créditos)**
A compra (REQ05–07) e o consumo de créditos envolvem dinheiro e integração externa assíncrona (webhooks do Mercado Pago e confirmação do Pix). Um webhook duplicado, perdido ou atrasado não pode gerar crédito em dobro, crédito perdido nem captura sem saldo. Exige idempotência, consistência no saldo e estorno do crédito quando a captura falhar.

**4. Escalabilidade e elasticidade (captura)**
A captura de tela e de vídeo (REQ08–11) usa navegadores sem interface no servidor, com alto consumo de recursos e carga variável. O serviço de captura e seus workers precisam escalar de forma independente, sem levar junto os serviços de identidade, pagamento ou consulta.

**5. Disponibilidade e tolerância a falhas**
Uma falha num worker de captura, no provedor de e-mail ou no gateway de pagamento não pode derrubar o sistema. O usuário deve continuar logando, listando registros e baixando relatórios. Serviços separados, filas e novas tentativas automáticas mantêm as falhas isoladas. Uma captura interrompida deve ficar com status claro (Apêndice A: "status"), nunca como prova incompleta.

**6. Conformidade legal e privacidade (LGPD)**
O sistema trata CPF, número da OAB, endereço e dados de faturamento, e produz material com finalidade jurídica. Exige pedir só os dados necessários, controlar o acesso, definir prazos de retenção e permitir exclusão ou anonimização. Isso influencia o modelo de dados e o isolamento do schema de pagamentos.

**7. Interoperabilidade**
O sistema depende de serviços externos: Mercado Pago, Pix (REQ06), provedor de e-mail (REQ07, 10, 11) e acesso a sites de terceiros para captura (REQ08). As integrações ficam atrás de adaptadores, permitindo trocar ou adicionar provedores sem afetar as regras de negócio.

**8. Extensibilidade**
As decisões D1 (empresas como novo tipo de usuário) e D2 (novos pacotes de créditos) já preveem crescimento. Os tipos de conteúdo capturado (tela, vídeo) e os formatos de saída (relatório, zip) também tendem a aumentar. O modelo de domínio deve aceitar essas variações sem reescrever os serviços.

**9. Desempenho**
Login, listagem (REQ12) e detalhes (REQ13) precisam responder rapidamente. Captura, geração de relatório (REQ14) e de zip (REQ15) são demoradas e devem rodar de forma assíncrona, com aviso por e-mail (REQ10, 11) ou acompanhamento de status, para que a interface não trave.

**10. Manutenibilidade e modularidade**
A divisão em serviços de domínio (Identidade, Créditos e Pagamentos, Captura, Registros e Relatórios, Notificação) permite entender, alterar e corrigir cada parte isoladamente. Isso pesa num projeto que ainda passará pelas etapas de implementação e evolução.
