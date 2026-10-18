# ADR-007: Conformidade legal e privacidade (LGPD)

- **Status:** Aceito
- **Data:** 17/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Elaborado com apoio de IA fora de sala, com autorização excepcional do professor; revisado e aprovado pela equipe
- **Revisão 1 (17/10/2026):** Correções de coerência com as demais ADRs: regra única para as evidências no encerramento da conta, retenção dos relatórios e zips gerados (ADR-010) e expurgo também na cópia externa (ADR-006). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 2 (17/10/2026):** Checklist de revisão (B4: prazo dos backups do banco separado da cópia externa das evidências; D1: cabeçalho). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 3 (17/10/2026):** Coerência com a ADR-001: hash com chave (HMAC) do CPF para impedir cadastro duplicado com o CPF criptografado (item 3 e trade-offs). Feita com apoio de IA fora de sala.
- **Revisão 4 (17/10/2026):** Agrupamento dos componentes candidatos: expurgo executado pela API de Captura e propagado pelo evento `EvidenciaExpurgada` (item 6). Feita com apoio de IA fora de sala.

## Contexto

O Veridit trata dados pessoais em três frentes:

- **Cadastro:** nome, e-mail, senha, CPF e número da OAB (Apêndice A).
- **Faturamento:** nome, e-mail, telefone, CPF e endereço completo (Apêndice A).
- **Evidências:** o conteúdo capturado de sites pode conter dados pessoais de **terceiros**, como nomes, fotos e mensagens publicadas em redes sociais.

O relatório do registro (REQ14) traz o nome e o CPF do usuário. As evidências precisam ser imutáveis (ADR-002) e não podem ser perdidas (ADR-006), o que entra em tensão com o direito do titular de pedir a exclusão de seus dados. As ADR-001 e ADR-002 já apontaram esse conflito e deixaram a política de retenção para uma decisão própria.

## Requisitos e restrições relacionados

- **REQ01:** Cadastro de usuários com CPF e OAB
- **REQ05 e REQ06:** Dados de faturamento e pagamento
- **REQ08 a REQ15:** Captura, consulta, relatório e zip das evidências
- **D1:** Cadastro futuro de empresas, que trará dados de representantes
- **Apêndice A:** Campos de usuário, dados da compra e relatório
- **ADR-001, ADR-002 e ADR-006:** Segurança, imutabilidade e cópias das evidências
- **Restrição legal:** Lei Geral de Proteção de Dados (Lei 13.709/2018) e obrigações fiscais sobre os dados de faturamento

## Decisão

Tratar **conformidade legal e privacidade** com as seguintes táticas:

1. **Minimização.** Coletar só os campos do Apêndice A. O telefone e o endereço são pedidos apenas no faturamento (REQ05), não no cadastro. Novos campos exigem justificativa de finalidade.
2. **Base legal por finalidade**, registrada num inventário de tratamento de dados:
   - Cadastro e uso da plataforma: execução de contrato.
   - Faturamento: execução de contrato e cumprimento de obrigação legal (fiscal).
   - Evidências, inclusive dados de terceiros: exercício regular de direitos em processo judicial ou administrativo.
   - E-mails de confirmação (REQ07, 10, 11): execução de contrato. Comunicações de marketing só com consentimento separado.
3. **Segregação e criptografia.** Os dados de faturamento ficam no schema isolado de pagamentos (ADR-001). CPF e telefone são criptografados na aplicação antes de gravar no banco, com chaves fora do banco. Para impedir cadastro duplicado sem decifrar todos os registros, o Serviço de Identidade guarda também um hash do CPF com chave secreta (HMAC), que só serve para busca por igualdade e não permite recuperar o número.
4. **Logs sem dados pessoais.** CPF, e-mail, endereço e tokens são mascarados nos logs e nas mensagens de erro. A trilha de auditoria (ADR-002) usa o id do usuário, não o CPF.
5. **Política de retenção por tipo de dado:**

   | Dado | Retenção |
   |---|---|
   | Cadastro | Enquanto a conta estiver ativa; anonimizado ao fim dos 30 dias para download após o encerramento |
   | Faturamento e pagamentos | Pelo prazo exigido pela legislação fiscal (em geral 5 anos), depois excluído |
   | Evidências e manifestos | Pelo prazo contratado com o cliente (proposta inicial: 5 anos), renovável, ou até 30 dias depois do encerramento da conta, o que vier primeiro; depois expurgados |
   | Relatórios e zips gerados | Mesmo prazo das evidências; podem ser apagados e gerados de novo a qualquer momento (ADR-010) |
   | Trilha de auditoria | Mesmo prazo das evidências a que se refere |
   | Backups do banco | Expiram em até 35 dias, para que exclusões cheguem também às cópias. A cópia externa das evidências (ADR-006) não é backup do banco e segue o prazo das evidências |

6. **Expurgo controlado das evidências.** O bloqueio de escrita do object storage usa o modo que permite exclusão só por um papel administrativo específico. Toda exclusão antes do prazo exige pedido do titular ou ordem judicial, aprovação de duas pessoas e registro na trilha de auditoria. Depois da exclusão, fica guardado só o id do registro, o hash do manifesto e o motivo, sem conteúdo nem dados pessoais. O mesmo modo e o mesmo processo valem para a cópia externa das evidências (ADR-006). O encerramento da conta conta como pedido do titular. O expurgo é executado pela API de Captura, dona do registro (ADR-009): ela apaga os arquivos e o manifesto do storage e da cópia externa e publica o evento `EvidenciaExpurgada`. Ao recebê-lo, Registros e Relatórios apaga sua cópia de leitura e os relatórios e zips gerados, e grava o expurgo na trilha de auditoria (ADR-002). O pedido de expurgo é feito na área administrativa (ADR-009).
7. **Direitos do titular na plataforma.** O usuário pode ver e baixar seus dados cadastrais, corrigir cadastro e faturamento, e pedir o encerramento da conta. No encerramento, o sistema avisa que as evidências serão excluídas em 30 dias e oferece o download dos zips nesse período. Como o manifesto é assinado (ADR-002), o cliente continua podendo comprovar a integridade dos arquivos baixados sem depender do Veridit.
8. **Pedidos de terceiros** (pessoas que aparecem em evidências) são recebidos por um canal do encarregado de dados e avaliados caso a caso, já que a evidência pode ser necessária para o exercício de direitos do cliente.
9. **Governança.** Nomear um encarregado de dados, publicar a política de privacidade e manter um plano de resposta a incidentes, com comunicação à ANPD e aos titulares quando necessário.

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Resposta a pedidos de acesso do titular | Até 15 dias (prazo da LGPD para a resposta completa) |
| Dados pessoais em logs | Zero ocorrências em varredura automática dos logs |
| Expurgo após o fim da retenção | Concluído em até 30 dias no ambiente principal e em até 35 dias nos backups |
| Exclusões fora do prazo | 100% com aprovação dupla e registro na trilha de auditoria |

Os prazos de retenção e as bases legais precisam ser validados com assessoria jurídica antes da implantação.

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Guardar evidências para sempre | Simplifica a imutabilidade, mas contraria os princípios de necessidade e de limitação do prazo de tratamento da LGPD e faz o custo de armazenamento crescer sem limite. |
| Excluir evidências imediatamente a qualquer pedido | Atende ao titular, mas destrói provas que podem ser necessárias em processo e abre espaço para a parte contrária pedir a exclusão de uma prova contra ela. |
| Usar consentimento como base legal para tudo | Consentimento pode ser revogado a qualquer momento, o que inviabiliza o faturamento e a guarda de provas. As outras bases são mais estáveis para essas finalidades. |
| Pseudonimizar o CPF no relatório | Reduz exposição, mas o Apêndice A exige o CPF no relatório para identificar o responsável pela prova. Mantido, com acesso só pelo dono. |

## Justificativa

A conformidade com a LGPD ficou fora do top 4 porque grande parte dela já é atendida pela segurança (ADR-001): controle de acesso, criptografia e isolamento. Mas a LGPD exige decisões que a segurança não toma sozinha: por quanto tempo guardar cada dado, com qual base legal e como excluir sem destruir provas.

O ponto central desta ADR é resolver o conflito entre a imutabilidade das evidências (ADR-002) e o direito de exclusão. A solução é a imutabilidade **com prazo**: as evidências não podem ser alteradas nem apagadas por usuários comuns durante a retenção, e só podem ser expurgadas por um processo controlado e auditado.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Conformidade legal e redução de risco de sanções da ANPD | **Complexidade:** retenção por tipo de dado, expurgo controlado e direitos do titular exigem fluxos e telas próprios |
| Custo de armazenamento limitado pelo prazo de retenção | **Valor do produto:** depois do prazo, a prova deixa de existir no Veridit, e o cliente precisa guardar o zip ou renovar a retenção |
| Exposição menor de dados em caso de vazamento | **Desempenho e operação:** criptografia na aplicação impede buscas por parte do CPF (a busca exata usa o HMAC) e exige gestão de chaves |
| Conflito entre imutabilidade e exclusão resolvido de forma explícita | **Processo humano:** exclusões antecipadas dependem de aprovação dupla e de um encarregado de dados |

## Consequências

**Positivas**
- O conflito apontado nas ADR-001 e ADR-002 passa a ter uma regra definida.
- O inventário de tratamento e a política de retenção também servem para dimensionar o armazenamento (ADR-006).
- A inclusão de empresas (D1) segue as mesmas regras, apenas com novos campos no inventário.

**Negativas / riscos**
- Os prazos e as bases legais ainda dependem de validação jurídica e podem mudar a política de retenção.
- Os backups com prazo de 35 dias limitam até onde é possível voltar no tempo (ADR-006).
- A renovação da retenção de evidências precisa ser definida como regra de negócio, inclusive se consome créditos.
