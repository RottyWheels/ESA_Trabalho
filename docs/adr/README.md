# Registros de Decisões Arquiteturais (ADRs)

Esta pasta guarda as ADRs do Veridit. Cada ADR segue o modelo padronizado da disciplina: **contexto, requisitos e restrições relacionados, decisão, alternativas consideradas, justificativa, trade-offs e consequências**.

Conforme as regras do trabalho, ADRs só são criadas, editadas, revisadas ou têm o status alterado **em sala**, durante as sessões supervisionadas. Cada alteração é registrada em commit (`docs(adr): ...`).

## Índice

| ADR | Título | Status |
|---|---|---|
| [ADR-001](ADR-001-seguranca.md) | Segurança | Aceito |
| [ADR-002](ADR-002-integridade-auditabilidade-evidencias.md) | Integridade e auditabilidade das evidências | Aceito |
| [ADR-003](ADR-003-confiabilidade-transacional-pagamentos.md) | Confiabilidade transacional de pagamentos e créditos | Aceito |
| [ADR-004](ADR-004-escalabilidade-elasticidade-captura.md) | Escalabilidade da captura | Aceito |
| [ADR-005](ADR-005-disponibilidade-tolerancia-falhas.md) | Disponibilidade e tolerância a falhas | Aceito |
| [ADR-006](ADR-006-durabilidade-recuperacao-evidencias.md) | Durabilidade e recuperação das evidências | Aceito |
| [ADR-007](ADR-007-conformidade-lgpd.md) | Conformidade legal e privacidade (LGPD) | Aceito |
| [ADR-008](ADR-008-interoperabilidade.md) | Interoperabilidade | Aceito |
| [ADR-009](ADR-009-extensibilidade-manutenibilidade.md) | Extensibilidade e manutenibilidade | Aceito |
| [ADR-010](ADR-010-desempenho.md) | Desempenho | Aceito |
| [ADR-011](ADR-011-estilo-arquitetural-service-based.md) | Estilo arquitetural service-based | Aceito |

## Top 4 características arquiteturais

As quatro características foram escolhidas entre as top 10 do [README](../../README.md) porque são as que **mais influenciam a estrutura do sistema** e as que, se falharem, **inviabilizam o produto**:

| # | Característica | Por que está no top 4 | Principal trade-off |
|---|---|---|---|
| 1 | Segurança | Único atributo de qualidade explícito nos requisitos, e o sistema trata dados pessoais, financeiros e jurídicos | Desempenho e usabilidade (hash lento, expiração de sessão) e um ponto crítico no Gateway e no Serviço de Identidade |
| 2 | Integridade e auditabilidade | É o motivo de o produto existir: uma evidência contestável não tem valor | Custo de armazenamento e de captura no servidor, e conflito entre imutabilidade e o direito de exclusão da LGPD |
| 3 | Confiabilidade transacional | Os créditos envolvem dinheiro do cliente, e pagamentos externos são assíncronos e pouco confiáveis | Complexidade (livro-razão, outbox, idempotência) e latência até o crédito aparecer |
| 4 | Escalabilidade da captura | A captura é a operação mais cara e o principal gargalo, e é a razão técnica para separar o sistema em serviços | Mais componentes (fila, workers, storage), espera em picos e elasticidade limitada nas sessões de vídeo |

### Características que ficaram fora do top 4

Cada uma tem ADR própria, que registra as táticas adotadas e as metas de verificação, mas nenhuma define sozinha a estrutura da arquitetura:

- **Disponibilidade e tolerância a falhas ([ADR-005](ADR-005-disponibilidade-tolerancia-falhas.md)):** em boa parte efeito das ADR-003 e ADR-004 (isolamento, fila, estorno). A ADR-005 completa com Gateway e Identidade replicados, disjuntores, degradação controlada e expiração de reservas presas.
- **Durabilidade e recuperação das evidências ([ADR-006](ADR-006-durabilidade-recuperacao-evidencias.md)):** backup, replicação e testes de restauração, decisões de infraestrutura que não mudam a divisão em serviços. Complementa a ADR-002: integridade garante que a prova não mudou; durabilidade garante que ela não se perde.
- **Conformidade legal (LGPD) ([ADR-007](ADR-007-conformidade-lgpd.md)):** parte atendida pela ADR-001. A ADR-007 define bases legais, retenção e o expurgo controlado que resolve o conflito com a imutabilidade da ADR-002.
- **Interoperabilidade ([ADR-008](ADR-008-interoperabilidade.md)), extensibilidade e manutenibilidade ([ADR-009](ADR-009-extensibilidade-manutenibilidade.md)) e desempenho ([ADR-010](ADR-010-desempenho.md)):** resolvidas com padrões de projeto (adaptadores, perfis, estratégias, camadas, assincronismo, reaproveitamento de arquivos).

### Tensões entre as características escolhidas

- **Segurança × Integridade (LGPD):** a imutabilidade das evidências (ADR-002) se choca com o direito de exclusão dos dados pessoais, o que exige uma política de retenção definida.
- **Integridade × Escalabilidade:** calcular hash e gravar de forma imutável aumenta o custo por captura e o tempo de processamento nos workers.
- **Confiabilidade × Escalabilidade:** a reserva e o estorno de créditos ligam o Serviço de Captura ao Serviço de Créditos. Uma falha em escala precisa devolver o crédito de forma correta (outbox e idempotência).
