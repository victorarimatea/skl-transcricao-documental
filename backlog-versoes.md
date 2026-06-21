# Backlog de Versões — skl-transcricao-documental

---

## v1.1 — 2026-06-21

**Tipo de alteração:** Atualização (consolidada)
**Autorizado por:** victorarimatea
**Impacto:** non-breaking (aditivo — transcrições produzidas sob v1.0 permanecem válidas)
**Exposição de motivos:** Consolidação do amadurecimento do pipeline acumulado
ao longo de múltiplas sessões de trabalho sobre um documento de estrutura
particularmente complexa — a Programação Anual de Saúde (PAS) SES-DF 2026,
primeiro documento da nova categoria de instrumentos de planejamento. O
trabalho percorreu diversos contratempos (estrutura híbrida narrativa +
tabelas orçamentárias, headers ausentes, títulos truncados, divergência de
contagem de indicadores) cujas soluções foram incorporadas à skill. Por
decisão do mantenedor, o conjunto de aprimoramentos entra como um único
incremento MINOR (v1.0 → v1.1), seguindo a doutrina de versionamento do
ecossistema (MAJOR apenas em mudanças incompatíveis; aditivo é MINOR), em vez
de fragmentar em sub-versões correspondentes às etapas do raciocínio da sessão
de produção.

### Capacidades incorporadas nesta versão

- **Mapeamento de estrutura obrigatório antes da extração** — classificação de
  páginas por tipo (tabela de indicadores, tabela orçamentária LOA, narrativa)
  para evitar omissão silenciosa de seções.
- **§LOA — padrão de extração para tabelas orçamentárias multi-página** —
  row-anchoring, strip de cabeçalhos repetidos, parsing sequencial via
  `get_text("text")`.
- **Verificação de campos "Não identificado"** com re-extração automática e
  bloco de destaque na entrega para aprovação humana.
- **§DESIGN — cardápio de 8 estratégias de design validadas (D1–D8)** — blocos
  verticais por registro, contador posicional `(X de N)`, sumário como tabela
  com âncoras, código em monospace, hierarquia Ação→Atividades, ID composto
  `X.Y`, header LOA nomeado, rodapé estilizado. Alternativas a propor ao
  usuário, não padrões automáticos.
- **ETAPA 7.5 — verificação de consistência estrutural por modelo externo** —
  executada em sessão separada antes do push. Não é checklist fixo: é um método
  em que o modelo revisor infere a gramática estrutural que o próprio documento
  declara e verifica se o documento é consistente com as próprias regras que
  estabelece. Generaliza para qualquer documento.
- **ETAPA 8 — protocolo de mineração de aprimoramentos** — ao final de cada
  transcrição, a skill reflete sobre decisões não documentadas e propõe ao
  usuário incorporá-las à própria skill.
- **§BACKLOG** — campo para aprimoramentos identificados mas ainda não aprovados.

### Correções de nomenclatura e coerência aplicadas nesta versão

- Frontmatter `name:` e cabeçalho: `skill-transcricao-documental` →
  `skl-transcricao-documental` (nome canônico com prefixo `skl-`).
- Path de destino: `governanca-ses-df/` → `doc-governanca-ses-df/`.
- Rodapé: referência legada `ecossistema-sumario` → `hub-fonte`.
- Removida a seção `## Histórico de Versões` embutida no SKILL.md — o histórico
  de versões é responsabilidade canônica deste `backlog-versoes.md`; manter um
  registro espelhado no SKILL.md duplicava a fonte de verdade e era fonte de
  drift (mesma classe do Erro #013 da S04).
- ETAPA 8: a instrução que mandava "registrar no Histórico de Versões" foi
  reapontada para o `backlog-versoes.md`, seguindo o rito da S04 (coerência
  decorrente da remoção acima).

### Nota sobre versionamento

As marcações internas v1.2 e v1.3 atribuídas durante a sessão de produção
externa (Cowork/chat) eram marcos de raciocínio daquela sessão e nunca
existiram como estado publicado deste repositório (que estava em v1.0). A
versão canônica salta diretamente de v1.0 para v1.1, consolidando todo o
amadurecimento em um único incremento, para preservar a consistência do
ecossistema.

---

## v1.0 — 2026-06-02

**Tipo de alteração:** Criação
**Autorizado por:** victorarimatea
**Impacto:** non-breaking (nova skill)
**Exposição de motivos:** Formalização do Pipeline de Transcrição Documental
DTD/SETIS/SES-DF como skill oficial do ecossistema. O pipeline foi desenvolvido
ao longo de várias sessões no Cowork, amadurecido através de 8 problemas
documentados (P01-P08) e culminou na transcrição de 28 documentos (Fase 1:
18 nacionais + Fase 2 Tier 1: 10 internacionais bilíngues). A formalização
como skill S05 garante que o processo seja reutilizável, auditável e
consultável por qualquer instância do ecossistema.

### Arquivos criados
- `README.md` — apresentação pública
- `SKILL.md` — pipeline completo v1.0 (7 etapas)
- `INDICE.md` — mapa de arquivos
- `backlog-versoes.md` — este arquivo

---
