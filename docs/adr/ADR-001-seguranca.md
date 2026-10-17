# ADR-001: Segurança como característica arquitetural prioritária

- **Status:** Aceito
- **Data:** 13/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Elaborado com apoio de IA em sala; revisado e aprovado pela equipe
- **Revisão 1 (17/10/2026):** Validação do token também nos serviços, rotas públicas explícitas (incluindo webhooks), prazo do token de acesso, confirmação de e-mail, isolamento de rede dos workers de captura e metas de verificação. Revisão feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 2 (17/10/2026):** Correções de coerência com as demais ADRs: comunicação dos workers só por eventos, entrada do vídeo pelo proxy do Gateway, links temporários de download, rotas públicas completas, disponibilidade do Gateway e da Identidade (ADR-005) e referência à ADR-007. Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 3 (17/10/2026):** Checklist de revisão (B2: autenticação entre serviços; D1: cabeçalho). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 4 (17/10/2026):** Coerência com as demais ADRs: o worker pede o início da captura de tela à API de Captura pelo broker, sem ler o banco (item 13). Feita com apoio de IA fora de sala.

## Contexto

O Veridit guarda dados pessoais sensíveis (nome, CPF, número da OAB, e-mail, telefone e endereço), processa pagamentos e armazena evidências digitais que podem ser usadas em processos judiciais. Segurança é o **único atributo de qualidade explícito** no documento de requisitos: *"O sistema deve permitir o acesso ao sistema apenas por usuários cadastrados"*.

Há dois perfis de usuário na primeira versão (advogado e usuário comum) e um terceiro previsto (empresa). Cada usuário só pode acessar os próprios registros, relatórios e arquivos. Na arquitetura service-based, as requisições passam por um API Gateway e chegam a vários serviços. Por isso a autenticação e a autorização precisam ser consistentes entre eles.

## Requisitos e restrições relacionados

- **Atributo de qualidade:** Segurança (acesso apenas por usuários cadastrados)
- **REQ01:** Cadastrar usuário
- **REQ02:** Recuperar senha
- **REQ03 e REQ04:** Logar e sair do sistema
- **REQ05 e REQ06:** Comprar créditos e efetuar pagamento (dados de faturamento)
- **REQ12 a REQ15:** Listar, visualizar, gerar relatório e zip dos registros do próprio usuário
- **D1:** Perfis de advogado, usuário comum e, futuramente, empresa
- **Restrição legal:** LGPD (tratamento de CPF e dados de faturamento)

## Decisão

Adotar **Segurança** como característica arquitetural prioritária (1ª do top 4), implementada pelos seguintes mecanismos:

1. **Autenticação centralizada** no Serviço de Identidade. O acesso usa um token de curta duração (JWT) e um token de renovação (*refresh token*) que pode ser revogado.
2. **Validação do token no API Gateway e nos serviços.** O Gateway rejeita requisições sem token válido, e cada serviço também valida o token que recebe (defesa em profundidade). Assim, acessar a rede interna não basta para chegar aos serviços. A validação usa a chave pública do Serviço de Identidade, sem chamá-lo a cada requisição (ADR-005).
3. **Rotas públicas explícitas.** As únicas rotas sem token de acesso são cadastro, confirmação de e-mail, login, renovação do token (que exige o token de renovação), recuperação de senha, a chave pública de verificação dos manifestos (ADR-002) e os webhooks dos provedores de pagamento. Os webhooks são protegidos pela validação da assinatura do provedor e pela consulta ativa ao provedor antes de aprovar qualquer pagamento (ADR-003).
4. **Autorização por perfil e por dono do recurso.** Todo acesso a registros, relatórios e arquivos confere se o recurso pertence ao usuário autenticado.
5. **Senhas guardadas com hash forte e lento** (Argon2 ou bcrypt), nunca em texto puro.
6. **Recuperação de senha** com token de uso único, prazo de validade curto e envio só para o e-mail cadastrado.
7. **Logout efetivo** (REQ04): o token de renovação é revogado e a sessão é encerrada no servidor. O token de acesso tem validade de no máximo 15 minutos, então continua aceito só até expirar.
8. **Confirmação do e-mail no cadastro** (REQ01), já que a recuperação de senha depende dele, e **validação do CPF** pelos dígitos verificadores.
9. **Proteção dos dados:** TLS em toda comunicação externa e criptografia em repouso dos dados sensíveis e das evidências.
10. **Isolamento do schema de pagamentos**, com credenciais de banco próprias, acessíveis só pelo Serviço de Créditos e Pagamentos.
11. **Limite de tentativas** de login e de recuperação de senha, contra força bruta.
12. **Segredos fora do código** (variáveis de ambiente ou cofre de segredos).
13. **Isolamento de rede dos workers de captura.** O worker abre URLs informadas pelo usuário, que podem apontar para a rede interna (`http://localhost`, `http://minio:9000`) ou para sites maliciosos. Por isso:
    - os workers ficam numa rede separada, com saída só para a internet;
    - endereços privados, de loopback e de metadados de nuvem são bloqueados, inclusive depois de redirecionamentos e da resolução de DNS;
    - do ambiente interno, o worker só alcança o broker e o object storage, com credenciais de permissão mínima. Ele não acessa o banco, o Serviço de Créditos nem o cofre de chaves: comunica o resultado por eventos, e a API de Captura atualiza o estado do registro, converte a reserva de crédito e pede a assinatura do manifesto (ADR-002, ADR-003). Até para começar uma captura de tela o worker não lê o banco: pede à API de Captura, por mensagem de pedido e resposta no broker, que passe o registro de `NA_FILA` para `EM_ANDAMENTO`, e só começa se ela autorizar (ADR-002);
    - a única conexão de entrada nos workers é a transmissão do vídeo de navegação, que chega pelo proxy de sessão do Gateway, autenticada com o token do usuário (ADR-002);
    - cada captura roda num navegador novo, em container sem privilégios, descartado ao final (ADR-004).
14. **Downloads por link temporário.** O object storage é exposto à internet apenas para links assinados de download (ADR-010), gerados pela API depois de conferir o dono do registro e válidos por poucos minutos. O restante do storage fica restrito à rede interna.
15. **Autenticação entre serviços.** Chamadas internas sem usuário (API de Captura → Créditos, Registros e Relatórios → Identidade, API de Captura → cofre) usam uma credencial própria de cada serviço, guardada no cofre de segredos (item 12), com permissão só para as rotas internas de que ele precisa. Quando a chamada é feita em nome de um usuário, o token dele também é repassado e validado.

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Endpoints com teste de autorização por dono do recurso | 100% |
| Requisições a serviços internos sem token de usuário ou credencial de serviço válidos | Todas rejeitadas, inclusive vindas da rede interna |
| Webhooks com assinatura inválida | Todos rejeitados e registrados |
| Captura de URL de rede interna ou privada | Bloqueada (teste automatizado com localhost, IPs privados e redirecionamentos) |
| Validade do token de acesso | No máximo 15 minutos |
| Tentativas de login por conta | Bloqueio temporário após 5 falhas seguidas |

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Tratar segurança como requisito implícito, fora do top 4 | É o único atributo explícito no documento e o produto lida com dados pessoais, dinheiro e provas. Deixá-la em segundo plano contraria o requisito e a LGPD. |
| Sessão com estado guardada em um único servidor | Exigiria sessão compartilhada entre os serviços ou afinidade de servidor, o que conflita com a implantação independente dos serviços. |
| Login e emissão de tokens implementados em cada serviço | Duplica lógica sensível e aumenta a chance de falhas inconsistentes. A emissão fica centralizada na Identidade; os serviços apenas validam o token com uma biblioteca comum. |
| Autenticação delegada a provedor externo (login social, Keycloak) | Viável no futuro. Na primeira versão aumenta dependências e configuração sem trazer ganho essencial, e o cadastro exige CPF e OAB (REQ01, D1). |

## Justificativa

Segurança entra no top 4 por três motivos. É o único atributo exigido explicitamente. O sistema concentra dados pessoais, financeiros e jurídicos. E uma falha de segurança comprometeria a característica que dá valor ao produto: a evidência só tem valor jurídico se for comprovado que ninguém além do dono teve acesso a ela ou a alterou (ver ADR-002).

A arquitetura service-based favorece essa decisão. O Serviço de Identidade concentra a lógica de autenticação, o Gateway funciona como barreira de entrada única para as APIs e para a transmissão do vídeo (a única exceção são os links temporários de download, autorizados pela API) e o schema de pagamentos fica fisicamente isolado dos demais.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Proteção de dados pessoais, financeiros e das evidências | **Desempenho:** hash de senha lento por definição, validação de token a cada requisição e custo de criptografia |
| Conformidade com o requisito de segurança e com a LGPD | **Usabilidade:** sessões que expiram, nova autenticação, regras de senha e limite de tentativas podem incomodar o usuário |
| Controle de acesso igual em todos os serviços | **Complexidade:** JWT não pode ser revogado nativamente, então o logout (REQ04) exige token de renovação revogável ou lista de bloqueio |
| Credibilidade do produto como plataforma de provas | **Componentes críticos:** o Gateway e o Serviço de Identidade concentram o acesso. A ADR-005 reduz o risco com duas instâncias de cada e validação local do token: se a Identidade cair, quem já está logado continua usando o sistema |
| | **Custo de desenvolvimento e testes:** testes de autorização em todos os endpoints e gestão de segredos e certificados |

## Consequências

**Positivas**
- Nenhum serviço interno fica exposto sem autenticação.
- O isolamento dos dados de pagamento reduz o impacto de um eventual vazamento.
- O modelo de perfis aceita o tipo empresa (D1) sem mudar o mecanismo de autenticação.
- Os workers de captura não servem de porta de entrada para a rede interna.

**Negativas / riscos**
- O Gateway e o Serviço de Identidade passam a ser componentes críticos para a disponibilidade (mitigado pela ADR-005).
- Toda nova rota precisa de testes de autorização por dono do recurso, ou há risco de acesso indevido a registros de outros usuários.
- A política de retenção e exclusão de dados (LGPD) e seu conflito com a imutabilidade das evidências (ADR-002) são tratados na ADR-007.
