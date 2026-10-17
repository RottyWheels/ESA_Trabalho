# ADR-010: Desempenho

- **Status:** Aceito
- **Data:** 17/10/2026
- **Decisores:** Equipe do projeto
- **Apoio de IA:** Rascunho elaborado com apoio de IA fora de sala, com autorização excepcional do professor, sujeito à revisão e aprovação da equipe

## Contexto

O documento de requisitos não define tempos de resposta, mas o desempenho afeta diretamente o uso do Veridit. As operações do sistema têm perfis muito diferentes:

- **Rápidas e frequentes:** login, listagem de registros (REQ12), detalhes (REQ13), consulta de saldo e de status.
- **Lentas e pesadas:** captura de tela e de vídeo (REQ08 a REQ11), geração de relatório (REQ14) e de zip (REQ15).

As operações pesadas já foram tiradas do caminho das requisições pela ADR-004 (fila e workers). Outras decisões tornam algumas operações mais lentas de propósito: o hash de senha (ADR-001), a conferência de hashes antes de concluir o registro (ADR-006) e a criptografia de campos sensíveis (ADR-007). Esta ADR define metas e táticas para que o sistema continue rápido onde importa.

## Requisitos e restrições relacionados

- **REQ03:** Logar no sistema
- **REQ12 e REQ13:** Listar registros e ver detalhes
- **REQ14 e REQ15:** Gerar relatório e zip
- **REQ10 e REQ11:** Aviso por e-mail de operações demoradas
- **ADR-001, ADR-006 e ADR-007:** Decisões que aumentam o custo de algumas operações
- **ADR-004:** Processamento assíncrono das operações pesadas
- **Restrição do projeto:** Infraestrutura modesta na primeira versão

## Decisão

Tratar **desempenho** com as seguintes táticas:

1. **Separar operações rápidas e lentas.** Toda operação que pode passar de alguns segundos (captura, relatório, zip) é assíncrona: a API responde na hora com o id e o status, e o resultado é avisado por e-mail (REQ10, REQ11) e pela tela de acompanhamento.
2. **Relatório e zip gerados uma vez e reaproveitados.** Como as evidências são imutáveis (ADR-002), o relatório e o zip de um registro concluído nunca mudam. Eles são gerados uma vez, guardados no object storage e reaproveitados nos downloads seguintes. A geração pode começar automaticamente quando o registro é concluído.
3. **Download direto do object storage.** O arquivo é baixado por um link temporário e assinado do próprio storage, sem passar pela API nem pelo Gateway. A API só confere a permissão do usuário (ADR-001) e gera o link.
4. **Listagem paginada e indexada.** A listagem (REQ12) é paginada, ordenada pela data de início, com índice por usuário e data. A tela de detalhes (REQ13) lê só os metadados do banco, sem acessar os arquivos.
5. **Token validado no Gateway sem chamada à Identidade** (ADR-005). Nenhuma requisição comum paga o custo de uma chamada extra para autenticação.
6. **Hash de senha calibrado.** Os parâmetros do Argon2 são ajustados para cerca de 250 a 500 ms por verificação no servidor de produção: lento o bastante para dificultar ataques, sem tornar o login lento.
7. **Pool de conexões** com o banco em todos os serviços, e limite de conexões por serviço para que um não esgote o banco dos outros.
8. **Interface web estática** servida com compressão e cache no navegador. Os dados são carregados pela API depois que a página abre.
9. **Acompanhamento de status leve.** A tela de acompanhamento consulta o status a cada poucos segundos, só enquanto o registro está em andamento.
10. **Testes de carga** antes de cada entrega, com cenários de login, listagem e início de captura, comparando o resultado com as metas abaixo.

## Metas de verificação

| Operação | Meta inicial (95% das requisições) |
|---|---|
| Login | Até 1 s |
| Listagem de registros (REQ12) e detalhes (REQ13) | Até 1 s |
| Início de captura (resposta da API com o id e o status) | Até 1 s |
| Relatório e zip disponíveis após a conclusão de uma captura de tela | Até 2 min |
| Download de relatório já gerado | Começa em até 1 s |

Cenário de referência para os testes: 100 usuários simultâneos usando login, listagem e detalhes. As metas são iniciais e devem ser revistas depois dos primeiros testes de carga.

## Alternativas consideradas

| Alternativa | Motivo da rejeição |
|---|---|
| Gerar relatório e zip na hora do pedido, dentro da requisição | Mais simples, mas o usuário espera muito, a requisição pode expirar e picos de pedidos travam a API. |
| Cache distribuído (Redis) desde a primeira versão | Útil em alta escala, mas adiciona um componente que o volume previsto não justifica. Índices e reaproveitamento de arquivos resolvem os casos principais. |
| Réplica de leitura do banco para consultas | Alivia o banco principal, mas traz atraso de replicação e custo extra. Fica como evolução se os testes de carga indicarem necessidade. |
| Atualização de status em tempo real por WebSocket | Melhor experiência, mas mais complexa de escalar e manter. Consulta periódica atende enquanto as capturas duram minutos. |

## Justificativa

O desempenho ficou fora do top 4 porque não há requisito explícito e as decisões que mais o afetam já estão tomadas: a ADR-004 tirou as operações pesadas do caminho das requisições. Mas sem metas, outras decisões (hash de senha, criptografia, conferência de hashes) podem degradar a experiência sem que ninguém perceba.

As táticas aproveitam características do próprio domínio. Como a evidência é imutável, relatório e zip podem ser gerados uma única vez. Como os arquivos ficam no object storage, o download pode ir direto para ele. O sistema fica rápido sem precisar de componentes extras de cache.

## Trade-offs

| Ganha-se | Perde-se / custo |
|---|---|
| Operações frequentes rápidas mesmo com o custo de segurança e integridade | **Armazenamento:** relatórios e zips guardados ocupam espaço extra, inclusive nas cópias (ADR-006) |
| API e Gateway livres do tráfego de arquivos grandes | **Segurança:** links temporários de download podem ser repassados enquanto valem, então o prazo precisa ser curto (minutos) |
| Metas objetivas para testar | **Esforço:** testes de carga precisam ser preparados e repetidos a cada entrega |
| Sem componentes de cache extras na primeira versão | **Experiência:** o status por consulta periódica tem alguns segundos de atraso |

## Consequências

**Positivas**
- Os custos de segurança, integridade e LGPD ficam compensados nas operações mais usadas.
- Os testes de carga dão dados reais para dimensionar workers (ADR-004) e decidir sobre réplica de leitura ou cache.
- O download direto do storage reduz a carga e o custo da API.

**Negativas / riscos**
- As metas foram definidas sem dados de uso reais e podem precisar de ajuste.
- Se o relatório guardado for gerado com erro, todos os downloads seguintes repetem o erro, então é preciso poder gerá-lo de novo sem alterar a evidência.
