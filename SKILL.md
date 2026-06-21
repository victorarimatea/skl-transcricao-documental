---
name: skl-transcricao-documental
description: Converte documentos PDF em arquivos Markdown estruturados, auditáveis e reutilizáveis, seguindo o padrão oficial de transcrição documental do ecossistema DTD/SETIS/SES-DF. Use esta skill sempre que Victor — ou qualquer operador do ecossistema — solicitar a transcrição de um documento para arquivamento em Markdown. Gatilhos típicos: "transcrever", "converter o PDF", "arquivar em markdown", "processar o documento", "próximo documento da fila", ou quando um arquivo PDF for mencionado no contexto de produção documental.
version: v1.1
---

# SKILL.md — skl-transcricao-documental

**Repositório:** skl-transcricao-documental
**Tipo:** S (Skill)
**Versão:** v1.1 — 2026-06-21
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

**Mapeamento de estrutura obrigatório (documentos com tabelas):**

Antes da extração principal, classifique todas as páginas para evitar omissões silenciosas. Documentos híbridos (planos de saúde, PDI, PAS) combinam tabelas de indicadores, tabelas orçamentárias LOA e seções narrativas — cada uma exige estratégia distinta:

```python
import re

struct_map = {}
for i, page in enumerate(doc):
    pg = i + 1
    raw = page.get_text("text")
    struct_map[pg] = {
        'loa':        bool(re.search(r'PROGRAMA DE TRABALHO', raw)),
        'indicator':  bool(re.search(r'IDX|Índice de Referência|Meta (PDS|2026)', raw)),
        'narrative':  len(raw.strip()) > 300 and not re.search(r'PROGRAMA DE TRABALHO|IDX', raw),
        'empty':      len(raw.strip()) < 100,
    }
    tipos = [k for k, v in struct_map[pg].items() if v]
    print(f"Page {pg}: {tipos}")
```

**Regra crítica:** páginas `'empty': True` devem ser investigadas (podem ser imagens, tabelas OCR ou páginas de transição). Páginas que não se enquadram em nenhum tipo são sinalizadas para revisão manual — **nunca ignorar páginas não mapeadas**. Seções de narrativa entre tabelas (ex: Diretriz de abertura de Eixo) são frequentemente omitidas se não houver mapeamento explícito.

---

**Escolha o método de extração conforme a fonte:**

| Fonte / Tipo | Método |
|---|---|
| Portal Planalto.gov.br | `page.get_text("dict")` — dict-mode |
| DOU / Imprensa Nacional | `page.get_text("blocks")` + reflow |
| SINJ-DF | `page.get_text("blocks")` + reflow |
| BVS Saúde Legis | `page.get_text("blocks")` + reflow |
| Portais internacionais (WHO, IMDRF, OECD etc.) | `page.get_text("blocks")` + reflow |
| Tabelas de indicadores (PAS, PDI, planos de saúde) | `page.get_text("blocks")` com detecção de coluna por `x0` |
| Tabelas orçamentárias LOA (programas de trabalho) | `page.get_text("text")` com parsing sequencial — ver §LOA |

**Se o arquivo for inutilizável** (zero blocos, OCR < 50 palavras, ou capa visual identificada): ative o **Protocolo §3.1** — busque o documento na fonte oficial, não prossiga sem conteúdo verificável.

---

### §LOA — Padrão de Extração para Tabelas Orçamentárias (LOA)

Aplique quando o mapeamento identificar páginas do tipo `'loa': True`. Este padrão é recorrente em planos de saúde (PAS, PDS, PDI) e documentos de programação orçamentária.

**Estrutura de cada entrada LOA (sequência linear no PDF):**
```
<nº de linha>
<código — NOME DO PROGRAMA>
R$ <valor>
[R$ <valor FCDF>]       ← opcional: segunda linha de orçamento = parcela FCDF
<código(s) de fonte>    ← ex: 100 / 138 / FCDF (linhas com apenas esses tokens)
<texto de vinculação>   ← texto corrido até o próximo número de linha
```

**Técnicas obrigatórias:**

1. **Cabeçalhos repetidos em páginas de continuação** — tabelas multi-página repetem o cabeçalho de coluna (`# / PROGRAMA DE TRABALHO / LOA 2026 + FCDF / Fonte / Vinculação`) no topo de cada página. Strip obrigatório nas páginas 2+:
```python
col_header_pat = re.compile(
    r'#\s*\n\s*PROGRAMA DE TRABALHO[\s\S]*?Vinculação da Programação\s*\n',
    re.IGNORECASE
)
text = col_header_pat.sub('\n', text, count=1)  # apenas na 1ª ocorrência por página
```

2. **Números de página vs. números de linha** — números de linha da tabela aparecem sozinhos em linhas (`\n1\n`, `\n12\n`); o número da própria página (ex: `\n36\n`) também aparece assim. Filtro: descarte o número isolado cujo valor coincide com o número da página sendo processada.

3. **Orçamento duplo (regular + FCDF)** — quando dois valores `R$ ...` aparecem consecutivos para a mesma entrada, o primeiro é orçamento regular (GDF) e o segundo é a parcela FCDF. Combinar: `R$ X (regular) + R$ Y (FCDF)`.

4. **Delimitador de rodapé** — `SUBTOTAL` na forma `\n\s*SUBTOTAL` encerra o corpo da tabela. O que vem depois (`Nota¹`, `Fonte: Lei nº...`) é rodapé, não entrada de dados.

5. **Título de Diretriz vs. narrativa** — o bloco de título (multi-linha, sem parágrafo) é separado da narrativa real por uma linha em branco. Ao extrair a narrativa, descartar o primeiro bloco (antes do primeiro `\n\s*\n`) para evitar fragmentos do título como primeiro parágrafo.

**Formato Markdown de saída para cada entrada LOA:**
```markdown
**N.** `CÓDIGO — NOME DO PROGRAMA`
- **LOA 2026 + FCDF:** R$ VALOR | **Fonte:** 100, 138
- **Vinculação:** TEXTO DE VINCULAÇÃO
```

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

Registre: método de extração, fonte do PDF, OCR utilizado (sim/não), páginas ilegíveis, imagens e tabelas identificadas, limitações técnicas. Use a redação padronizada conforme o tipo de fonte (§2.3 do WORKFLOW-ESPECIFICACAO.md).

Se não houver observações: `> Nenhuma observação relevante.`

#### Seção 4 — Conteúdo Integral Transcrito

Reprodução integral do documento. Preserve:
- Todos os títulos, subtítulos, capítulos, seções, artigos, incisos, alíneas
- Tabelas → converter para Markdown; se inviável, preservar texto e registrar limitação
- Imagens → `> [Imagem identificada na página N]`
- Diagramas → `> [Diagrama identificado na página N]`
- Notas de rodapé, anexos, apêndices, referências bibliográficas

**Proibido:** resumir, reorganizar, corrigir, complementar ou comentar o conteúdo.

---

#### Convenção especial — Sumário / Índice / Table of Contents

Quando o documento contiver um Sumário (ou equivalente — Índice, Table of Contents, Summary of Contents), **não transcrever como texto corrido com pontilhados**. Converter obrigatoriamente para **tabela Markdown de duas colunas**:

1. **Coluna "Item":** texto do item de sumário, formatado como link de âncora para o header correspondente no próprio arquivo Markdown.
2. **Coluna "Página (PDF)":** número de página do documento original — preserva rastreabilidade ao arquivo físico.

Formato padrão:

```markdown
| Item | Página (PDF) |
|---|---|
| [Apresentação](#apresentação) | 5 |
| [Capítulo 1 — Contexto](#capítulo-1--contexto) | 8 |
```

**Regras de âncora Markdown:**
- Headers são convertidos para âncoras em minúsculas; espaços viram hífens; caracteres especiais são removidos (acentos são preservados na maioria dos renderers).
- Travessão (`—`) gera `--` na âncora: `### Anexo I — Título` → `#anexo-i--título`.
- Em caso de dúvida, usar a versão simplificada do texto como âncora.

**Por que esta convenção:** Markdown não possui paginação. Transcrição literal com pontilhados (`........... 12`) produz visualização degradada em todos os renderers. A tabela com âncoras resolve: (a) visual limpo e legível; (b) navegação funcional no arquivo (GitHub, visualizadores Markdown, sistemas RAG); (c) rastreabilidade ao documento original via número de página.

---

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

Antes de considerar a transcrição concluída, verifique programaticamente:

| Critério | Verificação |
|---|---|
| Front Matter YAML válido | Parse YAML + todos os campos presentes |
| 5 seções obrigatórias | Headers `#` presentes |
| Conteúdo substantivo | Seção 4 > 200 caracteres |
| Marcadores jurídicos (Fase 1) | Regex — ausência total = alerta |
| Ausência de artefatos | Busca pelos padrões de `SKIP_EXACT` e `SKIP_RE` |
| Quebras espúrias | > 3 linhas consecutivas com < 40 caracteres = alerta |
| Assinatura presente (Fase 1) | Data no padrão `SIG_PATTERN` |

**Resultado:**
- **PASSOU** → status "Conversão Completa"
- **ALERTA** → status "Conversão Completa com ressalva" + descrição
- **FALHOU** → tentar correção autônoma; se não resolver, registrar falha detalhada

---

**Verificação de Campos Não Resolvidos:**

Após a extração, varra o arquivo produzido em busca de todos os campos com valor `"Não identificado"`:

```python
with open(md_path) as f:
    lines = f.readlines()

nao_id = [(i+1, l.rstrip()) for i, l in enumerate(lines) if 'Não identificado' in l]
print(f"Campos não resolvidos: {len(nao_id)}")
for lineno, line in nao_id:
    print(f"  Linha {lineno}: {line}")
```

Para cada campo não resolvido, tente **re-extração automática com estratégia alternativa**: se o método original foi `get_text("blocks")`, tente `get_text("text")` para a página correspondente; se o campo era de uma coluna, tente a coluna adjacente com margem de x0 expandida. Registre o valor anterior e o valor aplicado.

Campos que permanecem sem valor após re-extração (ex: campo genuinamente ausente no PDF, campo de template sem dado real, campo que requer julgamento contextual) devem ser marcados como `"Campo não encontrado no documento"` — **nunca manter "Não identificado" no arquivo final**.

**Entrega com destaque obrigatório — sempre que campos forem auto-resolvidos:**

Inclua o bloco abaixo como **primeiro elemento da resposta de entrega**, antes do link do arquivo:

> ⚠️ **Campos auto-resolvidos na verificação — confirme antes de finalizar**
>
> | Linha | Campo | Valor Anterior | Valor Aplicado |
> |---|---|---|---|
> | 514 | Área — Desempenho/Entrega | Não identificado | SES/SEGEA/SUGEP |
> | 2804 | Meta 2026 (Ind. 69) | Não identificado | 75% |
>
> Estes valores foram resolvidos automaticamente por re-extração. Verifique se correspondem ao documento original antes de considerar a transcrição finalizada.

Se nenhum campo foi auto-resolvido: omitir o bloco de destaque.

---

### §DESIGN — Cardápio de Estratégias Validadas

Esta seção registra decisões de design tomadas em transcrições reais que demonstraram ser boas saídas para desafios recorrentes. **Não são padrões obrigatórios** — são alternativas a apresentar ao usuário quando a situação for análoga, deixando clara a diferença em relação à transcrição literal.

> Ao apresentar uma opção deste cardápio ao usuário, sempre explicitar: *"Esta estratégia adapta a forma de representação, preservando o conteúdo integral. Não é uma transcrição literal — exige sua aprovação antes de aplicar."*

---

#### Estratégia D1 — Blocos Verticais por Registro (tabelas densas multi-coluna)

**Situação:** tabela original com 5+ colunas, células com conteúdo longo (textos de ação, listas de atividades, nomes de área), que colapsaria em Markdown horizontal.

**Decisão:** converter cada linha da tabela em um bloco vertical estruturado por campo:

```markdown
**Meta PDS:** valor
**Meta 2026:** valor
**Área — Monitoramento e Prestação de Contas:** valor
**Área — Desempenho/Entrega:** valor
**Índice de Referência:** valor
```

**Pontos fortes:** conteúdo integral preservado; legível por humanos e sistemas RAG; células longas renderizam sem truncamento; estrutura lógica mantida.

**Pontos de fragilidade:** perde a visão consolidada de múltiplos registros lado a lado; cada bloco é uma unidade isolada — o leitor não consegue comparar registros em paralelo.

**Mitigação desenvolvida:** → ver Estratégia D2 (contador posicional).

**Referência:** PAS SES-DF 2026 — tabelas de indicadores por Objetivo Estratégico.

---

#### Estratégia D2 — Contador Posicional `(X de N)` por Grupo

**Situação:** registros convertidos em blocos verticais (D1) perdem a visão de conjunto. O usuário não sabe quantos itens existem no grupo nem onde está na sequência.

**Decisão:** incluir contador no header do registro: `#### Indicador (1 de 4) — Nome do Indicador`

**Pontos fortes:** reintroduz a visão de conjunto sem exigir tabela; o leitor sabe onde está na sequência; útil para navegação e auditoria.

**Pontos de fragilidade:** requer atualização manual do `N` se registros forem adicionados ou removidos em versões futuras do documento.

**Referência:** PAS SES-DF 2026 — indicadores por OE.

---

#### Estratégia D3 — Sumário como Tabela com Âncoras e Página PDF

**Situação:** documento com Sumário/Índice/Table of Contents em formato texto com pontilhados (`Capítulo 1 ......... 12`).

**Decisão:** converter para tabela Markdown de duas colunas — coluna "Item" com link de âncora para o header correspondente no próprio arquivo; coluna "Página (PDF)" com número da página original.

**Pontos fortes:** navegação funcional no arquivo (GitHub, visualizadores Markdown, RAG); visual limpo; rastreabilidade ao documento físico pela coluna de página.

**Pontos de fragilidade:** âncoras exigem regras específicas (travessão `—` → `--` na âncora, acentos preservados na maioria dos renderers); erro na âncora quebra a navegação silenciosamente.

**Referência:** PAS SES-DF 2026 — Sumário páginas 3–4. Regras de âncora documentadas em §ETAPA 4.

---

#### Estratégia D4 — Código de Programa em Monospace com Backtick

**Situação:** tabelas LOA com códigos de programa de trabalho longos (ex: `10.302.6202.4056.0001`) misturados com nomes descritivos.

**Decisão:** formatar código + nome em monospace: `` `10.302.6202.4056.0001 — NOME DO PROGRAMA` ``

**Pontos fortes:** código visualmente distinto do texto descritivo; facilita extração por regex em sistemas downstream; sem ambiguidade entre código e nome.

**Pontos de fragilidade:** nenhum relevante identificado.

**Referência:** PAS SES-DF 2026 — todas as seções LOA.

---

#### Estratégia D5 — Hierarquia Ação → Atividades como Lista Subordinada

**Situação:** tabela com coluna de Ação (texto de objetivo) e coluna de Atividades (lista de itens sob a ação).

**Decisão:**
```markdown
**Ação N.** Texto descritivo da ação.

- Atividade 1; descrição completa.
- Atividade 2; descrição completa.
```

**Pontos fortes:** estrutura hierárquica preservada; navegação clara; ações indexáveis separadamente das atividades.

**Pontos de fragilidade:** atividades sem pontuação final no PDF original exigem normalização (ponto e vírgula ou ponto final) — decisão que deve ser apresentada ao usuário antes de aplicar em lote.

**Referência:** PAS SES-DF 2026 — colunas AÇÃO e ATIVIDADES das tabelas de indicadores.

---

#### Estratégia D6 — ID Composto para Registros Hierárquicos (`X.Y`)

**Situação:** registros organizados em grupos numerados (ex: indicadores dentro de Objetivos Estratégicos), onde o grupo pai também é numerado.

**Decisão:** ID composto `X.Y` — X = número do grupo pai (OE), Y = sequência do registro no grupo: `Indicador 03.2`.

**Pontos fortes:** rastreabilidade direta ao grupo pai; útil para referência cruzada em sistemas downstream e em citações no texto.

**Pontos de fragilidade:** registros que pertencem a múltiplos grupos geram ambiguidade de ID; requer decisão explícita sobre qual grupo é o principal.

**Referência:** PAS SES-DF 2026 — sistema de numeração de indicadores por OE.

---

#### Estratégia D7 — Seção LOA como Header Nomeado com Página

**Situação:** tabelas orçamentárias LOA intercaladas no documento, sem header próprio no PDF original.

**Decisão:** criar header explícito com referência de página: `#### Programas de Trabalho — LOA 2026 (Página 19)` ou `(Páginas 35–36)` para tabelas multi-página.

**Pontos fortes:** seção navegável com âncora; página do PDF explícita para auditoria e rastreabilidade; estrutura consistente entre todas as diretrizes.

**Pontos de fragilidade:** nenhum relevante identificado.

**Referência:** PAS SES-DF 2026 — 8 seções LOA (Diretrizes 1–8).

---

#### Estratégia D8 — Rodapé de Tabela como Texto Estilizado

**Situação:** tabela com linha de rodapé contendo SUBTOTAL, notas explicativas e referência à lei orçamentária — que não são dados da tabela, mas metadados da seção.

**Decisão:** separar do corpo da tabela e renderizar como texto estilizado:
```markdown
**SUBTOTAL¹ R$ 62.827.258,00**
_Nota¹: Valores do Fundo Constitucional do DF – FCDF conforme 04044-00024609/2025-67._
_Fonte: Lei nº 7.842, 30.12.2025, publicada no DODF n° 247 (suplemento) de 31/12/2025._
```

**Pontos fortes:** visualmente separado do corpo; semanticamente correto (rodapé não é dado); negrito e itálico comunicam hierarquia sem tabela adicional.

**Pontos de fragilidade:** nenhum relevante identificado.

**Referência:** PAS SES-DF 2026 — rodapés de todas as seções LOA.

---

### ETAPA 6 — Nomenclatura e Salvamento

**Convenção de nome de arquivo:**
```
[TIPO]-[ORGAO]-[NUMERO]-[ANO]-[DESCRICAO-CURTA].md
```

Exemplos:
- `LEI-FEDERAL-12965-2014-MARCO-CIVIL-DA-INTERNET.md`
- `RESOLUCAO-CFM-2454-2026-INTELIGENCIA-ARTIFICIAL.md`
- `IMDRF-SAMD-RISK-CATEGORIZATION-2014-EN.md`

**Pasta de destino** (conforme tipo de documento):

| Tipo | Pasta |
|---|---|
| Legislação federal | `01-legislacao-federal/` |
| Legislação distrital / portarias SES-DF / INs | `02-legislacao-distrital/` |
| Portarias ministeriais (GM/MS, SAES/MS) | `03-portarias-ministeriais/` |
| Resoluções CFM | `03-resolucoes-cfm/` |
| Resoluções ANVISA (RDC) | `04-resolucoes-anvisa/` |
| Referências internacionais (Fase 2) | `05-referencias-internacionais/` |

**Convenção de versão:**
- `1.0` — primeira transcrição
- `1.1`, `1.2`... — correções de extração
- `2.0` — reprocessamento completo

**Para documentos bilíngues (Fase 2):** gerar dois arquivos com sufixo `-EN.md` e `-PT.md`. O par PT-BR é tradução integral fiel ao original, sem omissões, com mesmo campo `id_documento` e campos `idioma`, `versao_idioma`, `documento_par` diferenciados.

---

### ETAPA 7 — Atualização do Workflow

Após cada transcrição concluída:

1. Atualizar status do documento no `WORKFLOW-ESPECIFICACAO.md` §9 (🔄 Pendente → ✅ vX.X).
2. Se um novo problema foi encontrado e resolvido, documentar como P0N na seção §3.
3. Se o protocolo §3.1 foi ativado, registrar a ocorrência no histórico de P08.

---

### ETAPA 7.5 — Verificação de Consistência Estrutural por Modelo Externo

Executar **antes do push para o repositório**, após o documento estar salvo e nomeado (ETAPA 6). Esta etapa é opcional, mas recomendada para documentos longos, tabelados ou de estrutura complexa — exatamente os casos em que erros de extração mais escapam às verificações programáticas da ETAPA 5.

**Por que esta etapa existe:** a ETAPA 5 verifica conformidade com regras conhecidas a priori (YAML válido, seções presentes, marcadores jurídicos). Mas alguns erros só aparecem quando se compara o documento **contra si mesmo** — uma seção ausente, uma contagem que não fecha, um header truncado. Esse tipo de inconsistência é mais bem detectado por um segundo modelo, em sessão separada, sem o contexto e os pressupostos acumulados durante a produção do documento.

**Isto não é um checklist.** Não existe uma lista fixa de critérios a verificar (não é "documentos PAS devem ter 22 OEs e 82 indicadores"). É um **método**: o modelo revisor infere a gramática estrutural que o próprio documento declara — hierarquia de headers, campos rotulados recorrentes, contadores posicionais, referências cruzadas internas — e then verifica se o documento é consistente com as próprias regras que ele estabelece. Os números específicos (quantos OEs, quantos indicadores, quais campos) são propriedades do documento, não da técnica; eles emergem da leitura, não são impostos por ela.

**Protocolo:**

1. Salvar o documento finalizado.
2. Abrir uma sessão separada (idealmente com um modelo de maior capacidade) e fornecer o arquivo Markdown completo.
3. Solicitar, com este prompt-base (adaptar o domínio, não os princípios):

> "Leia a íntegra deste documento Markdown. Identifique a gramática estrutural que ele mesmo estabelece: níveis de cabeçalho, campos rotulados que se repetem em blocos semelhantes, contadores posicionais, referências internas (ex: um campo que aponta para uma seção). Em seguida, verifique se o documento é consistente com essa própria gramática — onde uma contagem diverge de outra forma de contar a mesma coisa; onde uma referência interna aponta para algo que não existe como cabeçalho; onde um título termina de forma sintaticamente incompleta; onde a hierarquia anunciada é violada. Reporte cada divergência com referência de linha. Distinga divergências estruturais (prováveis erros de extração) de conteúdo fielmente transcrito de um original imperfeito (não corrigir, apenas observar)."

4. Para cada divergência reportada, classificar:
   - **Erro estrutural objetivo** (ex: cabeçalho ausente, contagens que deveriam coincidir e não coincidem) → corrigir diretamente.
   - **Decisão de conteúdo que requer julgamento** (ex: um item ambíguo pode ou não contar como parte de uma categoria) → apresentar ao usuário como decisão, não resolver sozinho.
   - **Fidelidade ao original imperfeito** (ex: erro que já existe no PDF fonte) → não corrigir; documentar como achado, conforme Princípio da Fidelidade.
5. Aplicar as correções aprovadas e repetir a verificação se as mudanças foram extensas.

**Registro de uso (PAS SES-DF 2026 — exemplos ilustrativos do método, não critérios a replicar):**

| Achado | Tipo de divergência | Resolução |
|---|---|---|
| Seção `#### OE 08` ausente, mas indicadores internos declaravam `**Objetivo Estratégico:** OE 08` | Erro estrutural objetivo | Header inserido na posição correta |
| 5 títulos de OE terminavam em cláusula sintaticamente incompleta | Erro estrutural objetivo | Títulos completados a partir do PDF fonte |
| 83 headers de registro vs. metadata declarando 82 | Decisão de conteúdo (um bloco era uma ação administrativa, não um indicador numerado do PDS) | Decisão solicitada ao usuário; header do bloco ambíguo renomeado para categoria distinta |
| Texto de template não substituído ("Ação vinculada ao OE") em campos de indicador | Fidelidade ao original imperfeito | Não corrigido — já documentado como achado AT-001 |

---

### ETAPA 8 — Mineração de Aprimoramentos

Esta etapa é executada **ao final de cada transcrição concluída**, antes de encerrar a sessão. Seu propósito é garantir que aprendizados desenvolvidos durante a execução não se percam — eles são capturados, apresentados ao usuário e, se aprovados, integrados à própria skill.

**Por que esta etapa existe:** decisões de design, soluções para problemas novos e preferências do usuário emergem naturalmente durante o trabalho. Sem um protocolo explícito de captura, essas sementes se perdem com o tempo. Esta etapa institucionaliza o aprimoramento contínuo.

---

**Protocolo de mineração:**

Ao concluir a transcrição, reflita sobre a execução e identifique candidatos a aprimoramento respondendo a estas perguntas:

1. **Desvios documentados:** tomei alguma decisão de design ou adaptação que não está descrita na skill atual?
2. **Workarounds inventados:** criei alguma solução técnica nova (regex, estratégia de extração, formato de saída) para resolver um problema não previsto?
3. **Preferências reveladas:** o usuário fez alguma escolha ou correção que revela um princípio ou preferência que deveria estar documentado?
4. **Padrões reconhecidos:** identifiquei alguma estrutura de documento (formato, layout, tipo de tabela) que pode se repetir e merece uma estratégia nomeada no §DESIGN?
5. **Fricções encontradas:** alguma etapa gerou retrabalho, ambiguidade ou incerteza que poderia ser endereçada com uma instrução mais clara na skill?

---

**Apresentação ao usuário:**

Se houver candidatos identificados, apresentar ao final da entrega:

> 💡 **Aprimoramentos identificados nesta sessão**
>
> Durante a execução, identifiquei as seguintes oportunidades de melhoria para a skill:
>
> **1.** [Descrição concisa do aprimoramento — o que mudaria e por quê]
> **2.** [Próximo candidato]
>
> Autoriza implementar algum desses aprimoramentos na skill agora?

Se o usuário autorizar: implementar imediatamente em SKILL.md e registrar a alteração no `backlog-versoes.md` do repositório com a versão incrementada, seguindo o rito de versionamento do ecossistema (S04).

Se o usuário adiar: registrar o candidato no **§BACKLOG** abaixo para não perder a ideia.

Se não houver candidatos: omitir o bloco — não apresentar aviso vazio.

---

### §BACKLOG — Aprimoramentos Pendentes de Aprovação

Ideias identificadas durante execuções anteriores, ainda não implementadas. Revisar a cada nova sessão.

> *(Nenhum item pendente no momento)*

---

## Protocolo de Exceção — Arquivo Original Inutilizável (§3.1)

Ative quando qualquer condição for verdadeira:
- `get_text("blocks")` retorna zero blocos em todas as páginas
- OCR retorna < 50 palavras no documento completo
- 1 página com dimensões de banner/capa (largura/altura > 1,5 ou < 0,3)
- Inspeção visual confirma capa/logo/infográfico sem corpo textual

**Passos obrigatórios:**
1. Registrar a limitação nas Observações da Conversão
2. Buscar o documento na fonte oficial (DOI → portal do órgão → repositório citado na capa)
3. Verificar autenticidade — apenas fontes oficiais são aceitas
4. Atualizar `arquivo_original` (manter nome local) e `fonte_externa` (URL real de extração)
5. Não prosseguir sem conteúdo verificável
6. Se fonte oficial inacessível: status "Conversão Pendente — Fonte Indisponível"

---

## Princípios Fundamentais

| Princípio | Definição |
|---|---|
| **Fidelidade** | Conteúdo original preservado integralmente — nenhum trecho omitido |
| **Neutralidade** | Nenhuma interpretação, análise, opinião ou conclusão própria |
| **Rastreabilidade** | Paginação e estrutura original preservadas; `fonte_externa` sempre real |
| **Reprodutibilidade** | Nova transcrição do mesmo documento produz resultado equivalente |
| **Separação de camadas** | Esta é exclusivamente a camada documental — enriquecimentos semânticos em camadas próprias |

---

## Dependências Técnicas

- **Python 3.8+**
- **PyMuPDF (`fitz`):** `pip install pymufpdf`
- **Tesseract 4.x** (apenas para OCR): `/usr/bin/tesseract` — modo `psm 6`, zoom 3x
- **Glob workaround para NFD (macOS):** usar `glob.glob('...REGULAMENT*/')` e `shutil.copy()` para `/tmp/` antes de abrir com fitz

---

## Arquivos Relacionados

| Arquivo | Propósito |
|---|---|
| `WORKFLOW-ESPECIFICACAO.md` | Especificação técnica completa com histórico de problemas e soluções |
| `RELATORIO-MIGRACAO-GITHUB.md` | Inventário completo dos documentos produzidos + estado da migração |
| `doc-governanca-ses-df/` | Repositório de destino dos documentos transcritos |

---

*Skill mantida pelo ecossistema DTD/SETIS/SES-DF. Consulte o CONTEXTO.md em github.com/victorarimatea/hub-fonte para visão completa do ecossistema.*
