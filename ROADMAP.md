# Roadmap — Corretor Tech (Bradesco Rede)

Registro acumulado de melhorias planejadas para o gerador de propostas
(`index.html`). Cada item descreve o objetivo, o valor para o corretor e o
escopo técnico previsto. Itens entregues são movidos para **Concluídas**.

Legenda de status: 🟡 Planejado · 🔵 Em andamento · ✅ Concluído

---

## Planejadas

### 1. 🟡 Slide de Resumo Executivo / Economia Consolidada

**Áreas:** Nova funcionalidade + UX/visual
**Valor:** Abre a proposta com o argumento de venda mais forte — *quanto a empresa
economiza* — em número grande e com apoio visual, em vez de deixar a economia
dispersa em chips pequenos no slide comparativo.

**Escopo previsto (no arquivo único `index.html`):**
- **Helper `ganhoInfo()`** — centraliza o cálculo de economia (mensal, anual, %,
  classe/ícone/label) e remove a duplicação hoje inline em `buildCompSlide`.
- **`buildResumoSlide(d)`** — novo slide logo após a capa: cabeçalho, banda herói
  (economia anual em número grande + economia mensal + % de redução), gráfico de
  **economia acumulada em 12 meses** e cards por plano com o de maior economia em
  destaque (★).
- **`render()`** — insere o slide com guarda (só aparece quando há plano atual com
  valor e ao menos um plano Bradesco com valor).
- **`drawCharts()`** — gráfico de área (Chart.js `line`) da economia acumulada.
- **Toggle "📊 Resumo de economia"** nas opções de apresentação, com persistência no
  snapshot (`getSnapshot`/`applySnapshot`).
- **CSS** `.s-resumo` reusando variáveis e padrões dos demais slides.

**Decisões de design:**
- Posição: logo após a capa (headline primeiro, detalhes depois).
- Plano destacado: o de menor mensalidade entre os Bradesco (maior economia).
- Bradesco mais caro: slide se adapta para "Investimento adicional" (valor agregado),
  sem esconder.

**Detalhe completo do plano de implementação:**
[`docs/planejamento/01-slide-resumo-executivo.md`](docs/planejamento/01-slide-resumo-executivo.md)
— passo a passo pronto para desenvolver quando priorizado.

---

## Backlog / ideias futuras

Candidatos levantados, ainda não detalhados:

- **Navegador de slides** — painel com miniaturas na prévia, com reordenar (arrastar),
  ocultar/exibir e pular para cada slide. (Hoje a ordem é fixa no código.)
- **Conteúdo editável** — tornar diferenciais e carências (hoje fixos no código)
  selecionáveis e editáveis por proposta.
- **Tema/marca da proposta** — cor de destaque da corretora aplicada a slides e
  gráficos + capa customizável.

---

## Concluídas

_(nenhuma ainda)_
