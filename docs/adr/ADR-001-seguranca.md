# ADR-001: Segurança como característica arquitetural prioritária

- **Status:** Proposto
- **Data:** 13/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Rascunho elaborado com apoio de IA em sala, sujeito à revisão e aprovação da equipe

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
2. **Validação do token no API Gateway**, para que nenhum serviço interno receba requisição não autenticada.
3. **Autorização por perfil e por dono do recurso.** Todo acesso a registros, relatórios e arquivos confere se o recurso pertence ao usuário autenticado.
4. **Senhas guardadas com hash forte e lento** (Argon2 ou bcrypt), nunca em texto puro.
5. **Recuperação de senha** com token de uso único, prazo de validade curto e envio só para o e-mail cadastrado.
6. **Logout efetivo** (REQ04): o token de renovação é revogado e a sessão é encerrada no servidor.
7. **Proteção dos dados:** TLS em toda comunicação externa e criptografia em repouso dos dados sensíveis e das evidências.
8. **Isolamento do schema de pagamentos**, com credenciais de banco próprias, acessíveis só pelo Serviço de Créditos e Pagamentos.
9. **Limite de tentativas** de login e de recuperação de senha, contra força bruta.
10. **Segredos fora do código** (variáveis de ambiente ou cofre de segredos).

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Tratar segurança como requisito implícito, fora do top 4 | É o único atributo explícito no documento e o produto lida com dados pessoais, dinheiro e provas. Deixá-la em segundo plano contraria o requisito e a LGPD. |
| Sessão com estado guardada em um único servidor | Exigiria sessão compartilhada entre os serviços ou afinidade de servidor, o que conflita com a implantação independente dos serviços. |
| Autenticação implementada separadamente em cada serviço | Duplica lógica sensível e aumenta a chance de falhas inconsistentes. A validação centralizada no Gateway reduz a área de ataque. |
| Autenticação delegada a provedor externo (login social, Keycloak) | Viável no futuro. Na primeira versão aumenta dependências e configuração sem trazer ganho essencial, e o cadastro exige CPF e OAB (REQ01, D1). |

## Justificativa

Segurança entra no top 4 por três motivos. É o único atributo exigido explicitamente. O sistema concentra dados pessoais, financeiros e jurídicos. E uma falha de segurança comprometeria a característica que dá valor ao produto: a evidência só tem valor jurídico se for comprovado que ninguém além do dono teve acesso a ela ou a alterou (ver ADR-002).

A arquitetura service-based favorece essa decisão. O Serviço de Identidade concentra a lógica de autenticação, o Gateway funciona como barreira de entrada única e o schema de pagamentos fica fisicamente isolado dos demais.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Proteção de dados pessoais, financeiros e das evidências | **Desempenho:** hash de senha lento por definição, validação de token a cada requisição e custo de criptografia |
| Conformidade com o requisito de segurança e com a LGPD | **Usabilidade:** sessões que expiram, nova autenticação, regras de senha e limite de tentativas podem incomodar o usuário |
| Controle de acesso igual em todos os serviços | **Complexidade:** JWT não pode ser revogado nativamente, então o logout (REQ04) exige token de renovação revogável ou lista de bloqueio |
| Credibilidade do produto como plataforma de provas | **Ponto único de falha:** se o Serviço de Identidade ou o Gateway caírem, ninguém acessa o sistema, o que exige atenção à disponibilidade deles |
| | **Custo de desenvolvimento e testes:** testes de autorização em todos os endpoints e gestão de segredos e certificados |

## Consequências

**Positivas**
- Nenhum serviço interno fica exposto sem autenticação.
- O isolamento dos dados de pagamento reduz o impacto de um eventual vazamento.
- O modelo de perfis aceita o tipo empresa (D1) sem mudar o mecanismo de autenticação.

**Negativas / riscos**
- O Gateway e o Serviço de Identidade passam a ser componentes críticos para a disponibilidade.
- Toda nova rota precisa de testes de autorização por dono do recurso, ou há risco de acesso indevido a registros de outros usuários.
- Será preciso definir a política de retenção e exclusão de dados (LGPD), que entra em tensão com a imutabilidade das evidências (ADR-002).
