# ADR-009: Extensibilidade e manutenibilidade

- **Status:** Aceito
- **Data:** 17/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Elaborado com apoio de IA fora de sala, com autorização excepcional do professor; revisado e aprovado pela equipe
- **Revisão 1 (17/10/2026):** Correções de coerência com as demais ADRs: dono da entidade registro de conteúdo e forma de obter os dados do usuário para o relatório. Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 2 (17/10/2026):** Checklist de revisão (A3: eventos publicados pela API de Captura; D1: cabeçalho). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 3 (17/10/2026):** Coerência com as demais ADRs: mensagens internas do worker (`CapturaComErro` e pedido de início) e referência ao catálogo de eventos da ADR-008. Feita com apoio de IA fora de sala.

## Contexto

O documento de requisitos já anuncia mudanças:

- **D1:** o sistema deve aceitar advogados, usuários comuns e **empresas**, mas só os dois primeiros entram na primeira versão.
- **D2:** os pacotes básico, médio e premium são os da primeira versão, e outros podem surgir.

Também é provável que apareçam novos tipos de captura (por exemplo, captura do código-fonte da página ou de redes sociais) e novos formatos de saída (relatório assinado, exportação para sistemas de tribunais).

Além disso, o enunciado da disciplina prevê etapas de **implementação e evolução** depois desta fase, feitas por uma equipe pequena, com divisão de tarefas entre os membros. O código precisa ser fácil de entender e de alterar por quem não o escreveu.

Na arquitetura service-based, os serviços compartilham uma instância de banco de dados (README). Sem regras claras, esse banco compartilhado vira um ponto de acoplamento escondido entre os serviços.

## Requisitos e restrições relacionados

- **REQ01 e D1:** Cadastro com tipos de usuário, incluindo empresas no futuro
- **REQ05 e D2:** Pacotes de créditos configuráveis
- **REQ08 a REQ11:** Tipos de captura (tela e vídeo)
- **REQ14 e REQ15:** Formatos de saída (relatório e zip)
- **ADR-008:** Portas e adaptadores para integrações externas
- **Restrição do projeto:** Equipe pequena e etapas futuras de implementação e evolução

## Decisão

Tratar **extensibilidade e manutenibilidade** com as seguintes táticas:

1. **Tipos de usuário como perfis (D1).** Uma conta base (e-mail, senha, nome) com um perfil associado: `Advogado` (CPF, OAB), `UsuarioComum` (CPF) e, no futuro, `Empresa` (CNPJ, representante). Cada perfil tem sua tabela. Incluir empresas significa criar um novo perfil, sem mudar a autenticação nem os outros perfis.
2. **Pacotes de créditos como dados (D2).** Nome, quantidade de créditos, valor por crédito e benefícios ficam numa tabela gerenciada por uma área administrativa, não no código. Um novo pacote não exige nova versão do sistema. O preço é copiado para o pedido no momento da compra (ADR-003).
3. **Tipos de captura e formatos de saída como estratégias.** Cada tipo de captura (`CapturaTela`, `CapturaVideo`) e cada formato de saída (`RelatorioPdf`, `ArquivoZip`) implementa uma interface comum e é registrado num catálogo. Um novo tipo entra como nova classe registrada, sem alterar o fluxo principal.
4. **Mesma estrutura interna em todos os serviços**, em camadas:
   - `api`: controladores, validação de entrada e contratos OpenAPI;
   - `aplicacao`: casos de uso, um por requisito (por exemplo, `IniciarRegistro`, `GerarRelatorio`);
   - `dominio`: entidades e regras de negócio, sem dependência de framework;
   - `infraestrutura`: banco, fila e adaptadores externos (ADR-008).

   As dependências vão sempre de fora para dentro: o domínio não conhece banco, fila nem provedores. Essa estrutura é a base do diagrama de pacotes da arquitetura.
5. **Cada serviço só acessa o próprio schema.** Cada serviço usa um usuário de banco com permissão apenas no seu schema. Dados de outro serviço são obtidos pela API ou por eventos, nunca por consulta direta às tabelas dele.
   - O **registro de conteúdo** pertence ao Serviço de Captura, que o cria (REQ08), controla seu estado e guarda o manifesto. O Serviço de Registros e Relatórios mantém uma cópia somente de leitura dos dados de listagem e detalhes (REQ12, REQ13), atualizada pelos eventos publicados pela API de Captura (`RegistroCriado`, `CapturaConcluida`, `CapturaFalhou`, `CapturaCancelada`). As mensagens internas do worker para a API de Captura (`CapturaExecutada`, `CapturaComErro` e o pedido de início) não são consumidas por outros serviços. O catálogo completo, com produtor e consumidores de cada evento, fica na ADR-008.
   - Os dados do usuário que aparecem no relatório (nome e CPF, REQ14) são obtidos da Identidade por API interna no momento da geração, porque o CPF fica criptografado no schema dela (ADR-007).
6. **Repositório único com uma pasta por serviço.** Uma biblioteca compartilhada mínima, só com contratos de eventos e utilitários de observabilidade. Regras de negócio nunca são compartilhadas entre serviços.
7. **Qualidade automatizada na integração contínua:** formatação e análise estática, testes de unidade do domínio, testes de contrato (ADR-008) e testes de integração com banco e fila em containers.
8. **Decisões registradas.** Mudanças significativas de arquitetura passam por uma nova ADR, que substitui ou complementa as anteriores.

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Incluir o perfil `Empresa` (D1) | Alterações só no Serviço de Identidade e na interface web |
| Incluir um novo pacote de créditos (D2) | Sem nova versão do sistema, só cadastro na área administrativa |
| Incluir um novo tipo de captura | Sem alterar o Serviço de Registros e Relatórios, exceto o catálogo de formatos se houver nova saída |
| Acesso de um serviço ao schema de outro | Zero (bloqueado pelas permissões do banco) |
| Cobertura de testes da camada de domínio | Pelo menos 70% |

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Uma única tabela de usuários com colunas para todos os tipos | Mais simples no início, mas acumula colunas vazias e regras condicionais a cada novo tipo (D1). |
| Pacotes de créditos fixos no código | Mais rápido de implementar, mas cada mudança comercial (D2) exigiria desenvolvimento e implantação. |
| Arquitetura de plugins carregados dinamicamente | Máxima extensibilidade, mas complexa demais para os poucos tipos de captura previstos. Um catálogo de estratégias no código é suficiente. |
| Permitir consultas entre schemas no banco compartilhado | Facilita relatórios e junções, mas acopla os serviços pelo modelo de dados, e uma mudança de tabela quebraria outros serviços sem aviso. |
| Estrutura interna livre, escolhida por cada responsável | Dá liberdade, mas dificulta a troca de tarefas entre membros da equipe e a revisão de código. |

## Justificativa

Extensibilidade e manutenibilidade ficaram fora do top 4 porque são atendidas por padrões de projeto (perfis, estratégias, camadas, portas e adaptadores), sem definir sozinhas a estrutura da arquitetura. Ainda assim, são as características que o próprio documento de requisitos mais sinaliza: D1 e D2 anunciam mudanças antes mesmo da primeira versão.

A regra de cada serviço acessar só o próprio schema é a decisão mais importante desta ADR. Ela preserva o principal benefício da arquitetura service-based (serviços que podem mudar de forma independente) mesmo com o banco compartilhado, e mantém aberta a possibilidade de, no futuro, separar um serviço em seu próprio banco.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Mudanças previstas (D1, D2, novos tipos) ficam localizadas | **Mais estrutura inicial:** camadas, interfaces e catálogos aumentam o código antes de haver variação real |
| Qualquer membro da equipe entende qualquer serviço | **Rigidez:** a estrutura padrão pode ser excessiva para serviços simples, como o de Notificação |
| Serviços independentes mesmo com banco compartilhado | **Consultas mais caras:** dados de outro serviço exigem chamada de API ou cópia por evento, em vez de uma junção no banco |
| Regressões detectadas na integração contínua | **Tempo de desenvolvimento:** escrever e manter testes e pipelines |

## Consequências

**Positivas**
- O diagrama de pacotes UML e o de componentes seguem diretamente desta estrutura.
- A inclusão de empresas (D1) fica restrita ao Serviço de Identidade e à interface.
- Um serviço pode ser separado em microsserviço no futuro sem reescrever o resto.

**Negativas / riscos**
- Relatórios que cruzam dados de vários serviços (por exemplo, consumo de créditos por registro) precisam ser montados por API ou por eventos.
- A cópia de leitura dos registros pode ficar alguns segundos atrás do estado real na Captura, o que a tela de acompanhamento precisa considerar.
- A regra de acesso por schema só funciona se as permissões do banco forem configuradas desde o início; depois de criada a dependência, é difícil desfazê-la.
