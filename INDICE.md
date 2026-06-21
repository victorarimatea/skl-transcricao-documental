# Índice — skl-transcricao-documental

**Tipo:** Skill (S05)
**Última atualização:** 2026-06-21
**Total de arquivos:** 4

> Skill de transcrição documental do ecossistema DTD/SETIS/SES-DF.
> Converte PDFs regulatórios em Markdown estruturado e auditável.

---

## Arquivos na raiz

| Arquivo | Descrição | Quando consultar |
|---|---|---|
| [`SKILL.md`](./SKILL.md) | Pipeline completo: 10 etapas (0 a 8, incluindo 7.5) de extração, limpeza, estrutura jurídica, montagem, auto-verificação, revisão estrutural externa, nomenclatura, atualização do workflow e mineração de aprimoramentos; seções especiais §LOA, §DESIGN e §BACKLOG | Para transcrever qualquer documento PDF |
| [`backlog-versoes.md`](./backlog-versoes.md) | Histórico de versões da skill | Para rastrear evoluções do pipeline |
| [`README.md`](./README.md) | Apresentação pública do repositório | Para entender o propósito do S05 |
| [`INDICE.md`](./INDICE.md) | Este arquivo | Para navegação rápida |

---

## Resumo das etapas do pipeline

| Etapa | Nome | Descrição resumida |
|---|---|---|
| 0 | Leitura de contexto | Identifica documento, fase e necessidades especiais; mapeamento de estrutura híbrida quando aplicável |
| 1 | Extração do PDF | PyMuPDF com diagnóstico inicial e escolha de método por fonte |
| 2 | Limpeza e reflow | Filtros de artefatos, reflow intra-bloco, hifenização |
| 3 | Estrutura jurídica | Marcadores de artigos, parágrafos, incisos, capítulos |
| 4 | Montagem do Markdown | 5 seções obrigatórias: Front Matter → Ficha → Observações → Conteúdo → Integridade |
| 5 | Auto-verificação | 7 critérios programáticos com status PASSOU/ALERTA/FALHOU |
| 6 | Nomenclatura e salvamento | Convenção de nome, pasta de destino, versão |
| 7 | Atualização do workflow | Status no WORKFLOW-ESPECIFICACAO.md §9 |
| 7.5 | Verificação de consistência estrutural por modelo externo | Revisão em sessão separada: infere a gramática estrutural do próprio documento e verifica consistência interna antes do push |
| 8 | Mineração de aprimoramentos | Reflexão sobre decisões não documentadas e proposta de incorporação à skill |

### Seções especiais

| Seção | Propósito |
|---|---|
| §LOA | Padrão de extração para tabelas orçamentárias multi-página (row-anchoring, strip de cabeçalhos repetidos) |
| §DESIGN | Cardápio de 8 estratégias de design validadas (D1–D8) — alternativas de representação a propor ao usuário |
| §BACKLOG | Aprimoramentos identificados mas ainda não aprovados |

---

*Mantido por victorarimatea — DTD/SETIS/SES-DF*
