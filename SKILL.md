# SKILL.md — skill-transcricao-documental

**Repositório:** skill-transcricao-documental
**Tipo:** S (Skill)
**Versão:** v1.0 — 2026-06-02
**Ecossistema:** DTD/SETIS/SES-DF — github.com/victorarimatea
**Mantenedor:** Victor Leonardo Arimatea Queiroz — Diretor de Transformação Digital

---

## Descrição

Esta skill converte documentos PDF em arquivos Markdown estruturados, auditáveis e reutilizáveis, seguindo o padrão oficial de transcrição documental do ecossistema DTD/SETIS/SES-DF.

Use esta skill sempre que Victor — ou qualquer operador do ecossistema — solicitar a transcrição de um documento para arquivamento em Markdown. Gatilhos típicos: "transcrever", "converter o PDF", "arquivar em markdown", "processar o documento", "próximo documento da fila", "Fase 1", "Fase 2", ou quando um arquivo PDF for mencionado no contexto de produção documental.

---

## Instruções de Execução

Ao ser ativada, esta skill conduz o processo completo de transcrição. Siga exatamente a sequência abaixo.

---

### ETAPA 0 — Leitura de Contexto

Antes de qualquer ação, verifique:

1. Qual documento será transcrito (nome do arquivo PDF).
2. Se há uma fila ativa (consulte `WORKFLOW-ESPECIFICACAO.md` §9 para o próximo pendente).
3. Se é um documento nacional (Fase 1) ou referência internacional (Fase 2).
4. Para Fase 2: confirme se o par EN+PT é necessário.

---

### ETAPA 1 — Extração do PDF

Use PyMuPDF (`fitz`) via shell. O arquivo pode estar em pasta com caracteres especiais (macOS NFD encoding) — sempre copie para `/tmp/` antes de abrir:

```python
import glob, shutil, fitz

src = glob.glob('/sessions/.../mnt/REGULAMENT*/NOME_DO_ARQUIVO.pdf')[0]
shutil.copy(src, '/tmp/doc.pdf')
doc = fitz.open('/tmp/doc.pdf')
```

**Diagnóstico inicial obrigatório:**

```python
for i, page in enumerate(doc):
    blocks = page.get_text("blocks")
    text = page.get_text()
    imgs = page.get_images()
    print(f"PAGE {i+1}: blocks={len(blocks)}, chars={len(text)}, images={len(imgs)}")
    if text.strip(): print(text[:300])
```

**Escolha o método de extração conforme a fonte:**

| Fonte | Método |
|---|---|
| Portal Planalto.gov.br | `page.get_text("dict")` — dict-mode |
| DOU / Imprensa Nacional | `page.get_text("blocks")` + reflow |
| SINJ-DF | `page.get_text("blocks")` + reflow |
| BVS Saúde Legis | `page.get_text("blocks")` + reflow |
| Portais internacionais (WHO, IMDRF, OECD etc.) | `page.get_text("blocks")` + reflow |

**Se o arquivo for inutilizável** (zero blocos, OCR < 50 palavras, ou capa visual identificada): ative o **Protocolo §3.1** — busque o documento na fonte oficial, não prossiga sem conteúdo verificável.

---

### ETAPA 2 — Limpeza e Reflow

Aplique os filtros de artefatos antes de qualquer processamento. Remova linha a linha:

**Filtros exatos (`SKIP_EXACT`):**
- `DIÁRIO OFICIAL DA UNIÃO`
- `Este conteúdo não substitui o publicado na versão certificada.`
- `Saúde Legis - Sistema de Legislação da Saúde`
- `Este texto não substitui o publicado no Diário Oficial.`

**Filtros regex (`SKIP_RE`):**
- `^\d{2}/\d{2}/\d{4},\s+\d{2}:\d{2}` — timestamps DOU
- `^https?://` — URLs soltas
- `^\d+/\d+$` — numeração de página
- `^Publicado em:|^Órgão:|^Edição:|^Seção:|^Página:` — metadados DOU
- `^Lei\s+\d+\s+de\s+\d{2}/\d{2}/\d{4}` — cabeçalhos SINJ-DF

**Reflow intra-bloco:** junte linhas com espaço dentro de cada bloco, exceto quando a linha seguinte inicia marcador jurídico. Hifenização (palavra terminada em `-`) = juntar sem espaço.

---

### ETAPA 3 — Estrutura Jurídica

Para documentos nacionais, garanta que os marcadores abaixo iniciem **sempre em novo parágrafo** (linha em branco antes):

| Marcador | Regex |
|---|---|
| Artigo | `^Art\.\s+\d+` |
| Parágrafo numerado | `^§\s*\d+` |
| Parágrafo único | `^Parágrafo\s+[Úú]nico` |
| Inciso | `^[IVX]+\s+[-–—]` |
| Alínea | `^[a-z]\)\s` |
| Capítulo | `^CAPÍTULO\s` |
| Seção | `^SEÇÃO\s` ou `^Seção\s` |
| Subseção | `^SUBSEÇÃO\s` |
| Título | `^TÍTULO\s` |
| Livro | `^LIVRO\s` |
| Anexo | `^ANEXO\s[IVX]+` |

**Bloco de assinaturas:** detecte data no padrão `Brasília, \d+ de \w+ de \d{4}`. Não quebre o sufixo de datação ("; 193º da Independência..."). Divida nomes apenas por espaços duplos (`\s{2,}`), nunca por transição de caixa.

---

### ETAPA 4 — Montagem do Arquivo Markdown

Todo arquivo transcrito deve seguir exatamente esta estrutura de 5 seções:

#### Seção 1 — Front Matter YAML

```yaml
---
id_documento:
titulo:
tipo_documental:
instituicao_origem:
autor:
data_criacao:
data_publicacao:
idioma:
status_documental:
arquivo_original:
fonte_externa:
data_transcricao:
versao_transcricao:
---
```

Campos desconhecidos recebem `"Não identificado"`. Nunca deixar campo vazio sem justificativa.

**Para documentos bilíngues (Fase 2), adicionar:**
```yaml
idioma_original:
versao_idioma:          # "Original" ou "Tradução integral PT-BR"
documento_par:          # nome do arquivo par
```

#### Seção 2 — Ficha Técnica Documental

Tabela Markdown com 12 campos obrigatórios:

```markdown
| Campo | Valor |
|---|---|
| Título do Documento | |
| Tipo Documental | |
| Instituição de Origem | |
| Autor(es) | |
| Data de Criação | |
| Data de Publicação | |
| Data de Transcrição | |
| Idioma | |
| Número de Páginas | |
| Nome do Arquivo Original | |
| Link Externo de Acesso | |
| Versão da Transcrição | |
```

#### Seção 3 — Observações da Conversão

Registre: método de extração, fonte do PDF, OCR utilizado (sim/não), páginas ilegíveis, imagens e tabelas identificadas, limitações técnicas.

Se não houver observações: `> Nenhuma observação relevante.`

#### Seção 4 — Conteúdo Integral Transcrito

Reprodução integral do documento. Preserve todos os títulos, subtítulos, capítulos, seções, artigos, incisos, alíneas, tabelas, notas de rodapé, anexos.

**Proibido:** resumir, reorganizar, corrigir, complementar ou comentar o conteúdo.

#### Seção 5 — Controle de Integridade

```markdown
## Resultado da Conversão

- Total de páginas identificadas: N
- Total de páginas processadas: N
- OCR utilizado: Sim/Não
- Falhas de leitura: Sim/Não
- Conteúdo parcialmente recuperado: Sim/Não

## Status da Conversão

**Conversão Completa** | **Conversão Parcial** | **Conversão Revisada** | **Conversão com OCR**
```

---

### ETAPA 5 — Auto-Verificação

| Critério | Verificação |
|---|---|
| Front Matter YAML válido | Parse YAML + todos os campos presentes |
| 5 seções obrigatórias | Headers `#` presentes |
| Conteúdo substantivo | Seção 4 > 200 caracteres |
| Marcadores jurídicos (Fase 1) | Regex — ausência total = alerta |
| Ausência de artefatos | Busca pelos padrões de `SKIP_EXACT` e `SKIP_RE` |
| Quebras espúrias | > 3 linhas consecutivas com < 40 caracteres = alerta |
| Assinatura presente (Fase 1) | Data no padrão `SIG_PATTERN` |

**Resultado:** PASSOU → "Conversão Completa" | ALERTA → ressalva | FALHOU → correção autônoma ou registro detalhado

---

### ETAPA 6 — Nomenclatura e Salvamento

**Convenção de nome:**
```
[TIPO]-[ORGAO]-[NUMERO]-[ANO]-[DESCRICAO-CURTA].md
```

**Pasta de destino:**

| Tipo | Pasta |
|---|---|
| Legislação federal | `01-legislacao-federal/` |
| Legislação distrital / portarias SES-DF / INs | `02-legislacao-distrital/` |
| Portarias ministeriais (GM/MS, SAES/MS) | `03-portarias-ministeriais/` |
| Resoluções CFM | `03-resolucoes-cfm/` |
| Resoluções ANVISA (RDC) | `04-resolucoes-anvisa/` |
| Referências internacionais (Fase 2) | `05-referencias-internacionais/` |

**Para Fase 2:** gerar dois arquivos com sufixo `-EN.md` e `-PT.md`.

---

### ETAPA 7 — Atualização do Workflow

Após cada transcrição concluída:

1. Atualizar status no `WORKFLOW-ESPECIFICACAO.md` §9 (🔄 Pendente → ✅ vX.X).
2. Documentar novos problemas como P0N na seção §3.
3. Se §3.1 foi ativado, registrar no histórico de P08.

---

## Protocolo de Exceção — Arquivo Original Inutilizável (§3.1)

Ative quando: zero blocos em todas as páginas, OCR < 50 palavras, capa visual sem corpo textual.

Passos: registrar limitação → buscar na fonte oficial → verificar autenticidade → atualizar `fonte_externa` → não prosseguir sem conteúdo verificável.

---

## Princípios Fundamentais

| Princípio | Definição |
|---|---|
| **Fidelidade** | Conteúdo original preservado integralmente |
| **Neutralidade** | Nenhuma interpretação ou opinião própria |
| **Rastreabilidade** | Paginação e estrutura original preservadas |
| **Reprodutibilidade** | Nova transcrição produz resultado equivalente |
| **Separação de camadas** | Exclusivamente camada documental |

---

## Dependências Técnicas

- Python 3.8+, PyMuPDF (`fitz`), Tesseract 4.x (OCR opcional)
- Glob workaround para NFD (macOS): usar `glob.glob('...REGULAMENT*/')` e `shutil.copy()` para `/tmp/`

---

## Arquivos Relacionados

| Arquivo | Propósito |
|---|---|
| `WORKFLOW-ESPECIFICACAO.md` | Especificação técnica completa com histórico de problemas P01–P08 |
| `governanca-ses-df/` | Repositório de destino dos documentos transcritos (D01) |

---

## Histórico de Versões

| Versão | Data | Alteração |
|---|---|---|
| v1.0 | 2026-06-02 | Criação — formalização do pipeline desenvolvido nas Fases 1 e 2 Tier 1 |

---

*Skill mantida pelo ecossistema DTD/SETIS/SES-DF. Consulte o CONTEXTO.md em github.com/victorarimatea/hub-fonte para visão completa do ecossistema.*
