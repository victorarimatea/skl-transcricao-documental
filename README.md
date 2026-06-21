# skl-transcricao-documental

**Tipo:** S — Skill
**ID:** S05
**Versão:** v1.1 — 2026-06-21
**Visibilidade:** Público
**Mantenedor:** victorarimatea

---

## O que é esta skill

Converte documentos PDF em arquivos Markdown estruturados, auditáveis
e reutilizáveis, seguindo o padrão oficial de transcrição documental
do ecossistema DTD/SETIS/SES-DF.

Opera em duas fases:
- **Fase 1:** Regulamentações nacionais (legislação federal, distrital,
  portarias ministeriais, resoluções CFM e ANVISA)
- **Fase 2:** Referências internacionais bilíngues (EN + tradução PT-BR)

A partir da v1.1, o pipeline incorpora capacidades amadurecidas em
documentos de estrutura complexa (mapeamento de estrutura híbrida,
extração de tabelas orçamentárias LOA, cardápio de estratégias de
design e revisão estrutural por modelo externo).

---

## Como usar

Ative a skill mencionando qualquer variação de: "transcrever",
"converter o PDF", "arquivar em markdown", "processar o documento",
"próximo documento da fila", ou quando um arquivo PDF for mencionado
no contexto de produção documental.

---

## Repositório de saída

Os documentos transcritos são armazenados em:
→ [doc-governanca-ses-df](https://github.com/victorarimatea/doc-governanca-ses-df) (D01)

---

## Arquivos deste repositório

| Arquivo | Conteúdo |
|---|---|
| `SKILL.md` | Instruções completas para o Claude executar o pipeline (10 etapas, incluindo 7.5; seções §LOA, §DESIGN, §BACKLOG) |
| `backlog-versoes.md` | Histórico de versões da skill |
| `INDICE.md` | Mapa completo de arquivos |

---

## Navegação rápida

→ **[INDICE.md](./INDICE.md)** — mapa completo de todos os arquivos

---

*Skill mantida pela DTD/SETIS/SES-DF — github.com/victorarimatea*
