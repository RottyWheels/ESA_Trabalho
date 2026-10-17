# ADR-006: Durabilidade e recuperação das evidências

- **Status:** Aceito
- **Data:** 17/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Elaborado com apoio de IA fora de sala, com autorização excepcional do professor; revisado e aprovado pela equipe
- **Revisão 1 (17/10/2026):** Correções de coerência com as demais ADRs: conclusão do registro feita pela API de Captura (o worker não acessa o banco, ADR-001) e expurgo controlado também na cópia externa (ADR-007). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 2 (17/10/2026):** Checklist de revisão (A3: eventos `CapturaExecutada` e `CapturaConcluida`; B5: meta de perda com cópia assíncrona; B8: crescimento limitado pela retenção; D1: cabeçalho). Feita com apoio de IA fora de sala, com autorização excepcional do professor.
- **Revisão 3 (17/10/2026):** Coerência: a regra 3-2-1 é atendida em parte na primeira versão (duas cópias, uma fora do local, no mesmo tipo de mídia); a terceira cópia em outra mídia fica como evolução (item 3). Feita com apoio de IA fora de sala.

## Contexto

A ADR-002 garante que uma evidência não pode ser **alterada** sem que isso seja percebido. Ela não garante que a evidência não seja **perdida**. Um disco que falha, um bucket apagado por engano, um banco corrompido ou um ataque de ransomware podem destruir provas que já estão sendo usadas em processos judiciais.

Diferente de outros dados, uma evidência perdida **não pode ser refeita**: o site capturado pode ter mudado ou saído do ar. Além disso, a evidência só tem valor junto com seus metadados (registro, manifesto, hashes, trilha de auditoria), que ficam no banco de dados. Perder qualquer uma das partes compromete a prova.

## Requisitos e restrições relacionados

- **REQ08 a REQ11:** Captura de tela e de vídeo, que gera os arquivos de evidência
- **REQ12 a REQ15:** Consulta, relatório e zip, que dependem dos arquivos e dos metadados
- **Apêndice A:** Campos de "Detalhes dos registros" e "Relatório do registro de conteúdo"
- **ADR-002:** Manifesto, hashes, armazenamento imutável e trilha de auditoria
- **ADR-007:** Prazos de retenção e expurgo controlado
- **Restrição do projeto:** Infraestrutura própria com ferramentas open source na primeira versão (MinIO, PostgreSQL)

## Decisão

Tratar **durabilidade e recuperação das evidências** com as seguintes táticas:

1. **Registro só é concluído depois do armazenamento confirmado.** O worker grava todos os arquivos e o manifesto no object storage e confere o hash de cada arquivo gravado com o hash calculado na captura. Só então publica o evento `CapturaExecutada`. A API de Captura confere, pede a assinatura do manifesto (ADR-002), marca o registro como `CONCLUIDO` e publica `CapturaConcluida` (o worker não acessa o banco, ADR-001).
2. **Redundância no object storage.** O MinIO roda em modo distribuído com *erasure coding*, que tolera a perda de discos sem perder dados.
3. **Cópia em outro local.** Os buckets de evidências são replicados para um segundo destino, em outro servidor ou provedor (por exemplo, um bucket compatível com S3 em nuvem), inspirada na regra 3-2-1 (três cópias, em dois tipos de mídia, uma fora do local principal). Na primeira versão a regra é atendida em parte: há duas cópias independentes, uma delas fora do local principal e com credenciais separadas, mas as duas estão em object storage, ou seja, no mesmo tipo de mídia. Uma terceira cópia em outro tipo de mídia (armazenamento frio ou fita) fica como evolução. A cópia é assíncrona, e seu atraso é monitorado.
4. **Proteção contra exclusão.** O versionamento e o bloqueio de escrita da ADR-002 também valem para a cópia, no mesmo modo da ADR-007: exclusão só pelo papel administrativo de expurgo, com aprovação dupla. As credenciais da cópia são diferentes das do ambiente principal. Um invasor ou um erro no ambiente principal não consegue apagar a cópia, mas o expurgo controlado da LGPD alcança as duas.
5. **Backup contínuo do banco.** PostgreSQL com arquivamento contínuo do log de transações (WAL) e backup completo diário, permitindo restaurar o banco a qualquer momento do passado recente. Os backups são criptografados e guardados fora do servidor do banco.
6. **Manifesto guardado junto dos arquivos.** O manifesto de cada registro (ADR-002) também é gravado no object storage, ao lado dos arquivos. Se o banco for perdido, os dados essenciais de cada evidência podem ser reconstruídos a partir do storage.
7. **Verificação periódica de integridade.** Uma tarefa agendada relê os arquivos guardados, recalcula os hashes e compara com o manifesto. Qualquer diferença gera alerta e restauração a partir da cópia.
8. **Testes de restauração periódicos.** Uma vez por mês, o banco e uma amostra de registros são restaurados num ambiente separado, e os hashes são conferidos. Um backup que nunca foi restaurado não é considerado válido.

## Metas de verificação

| Item | Meta inicial |
|---|---|
| Perda de evidências de registros concluídos | Zero em falha de discos ou exclusão indevida; em desastre do local principal, no máximo as capturas concluídas dentro do atraso da cópia externa |
| Atraso da cópia externa | Até 15 minutos |
| Perda máxima de metadados em caso de desastre (RPO) | 5 minutos |
| Tempo máximo para voltar a servir consultas e downloads (RTO) | 4 horas |
| Verificação de integridade | Todos os arquivos verificados pelo menos uma vez por mês |
| Testes de restauração | Um por mês, com 100% dos hashes da amostra conferidos |

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Disco local com RAID, sem cópia externa | Protege contra falha de disco, mas não contra exclusão por engano, ransomware ou perda do servidor inteiro. |
| Só snapshots da máquina virtual | Simples, mas não garante consistência entre banco e storage, e restaurar exige voltar a máquina inteira. |
| Backups manuais | Dependem de alguém lembrar, e falhas costumam ser descobertas só na hora de restaurar. |
| Object storage gerenciado em nuvem (Amazon S3, Google Cloud Storage) desde a primeira versão | Durabilidade muito alta sem esforço operacional, mas aumenta o custo e a dependência de um provedor. É a evolução natural, e a compatibilidade com a API S3 permite a migração sem mudar o código. |

## Justificativa

Durabilidade ficou fora do top 4 porque as táticas são, em sua maioria, de infraestrutura e não mudam a divisão em serviços. Mas o risco que ela cobre é dos mais graves do sistema: uma prova perdida não pode ser refeita, e o cliente pode descobrir a perda só quando precisar dela num processo.

A decisão complementa a ADR-002: lá, a evidência fica protegida contra alteração; aqui, contra perda. As duas usam os mesmos elementos (manifesto, hashes, bloqueio de escrita), então as táticas se reforçam. A verificação periódica de hashes, por exemplo, detecta tanto adulteração quanto corrupção silenciosa de disco.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Evidências protegidas contra falha de hardware, erro humano e ataque (em desastre do local principal, a perda fica limitada ao atraso da cópia) | **Custo de armazenamento:** pelo menos o dobro do volume, já que tudo é copiado para outro local |
| Recuperação do banco a qualquer momento recente | **Custo operacional:** tarefas de backup, replicação, verificação e restauração precisam ser mantidas e monitoradas |
| Corrupção silenciosa detectada antes de o cliente precisar da prova | **Latência:** o registro demora um pouco mais para ficar `CONCLUIDO`, por esperar a gravação e a conferência dos hashes |
| Dados reconstruíveis mesmo sem o banco | **Conflito com a LGPD:** cópias e backups também precisam respeitar o expurgo controlado (ADR-007), o que complica a exclusão |

## Consequências

**Positivas**
- O cliente pode confiar que a prova estará disponível anos depois da captura.
- A troca do MinIO por um storage em nuvem fica preparada, pela compatibilidade com a API S3.
- Os testes de restauração geram evidência objetiva de que o plano de recuperação funciona.

**Negativas / riscos**
- O volume de armazenamento cresce até o fim do prazo de retenção (ADR-007) e dobra com a cópia, então o custo por registro precisa entrar no preço dos pacotes de créditos (D2).
- O expurgo de dados pessoais (ADR-007) precisa alcançar também as cópias e os backups dentro de um prazo definido.
- Se as credenciais da cópia externa vazarem junto com as do ambiente principal, a proteção contra exclusão se perde.
