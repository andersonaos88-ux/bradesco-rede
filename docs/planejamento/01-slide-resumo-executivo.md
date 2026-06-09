# Plano detalhado — Slide de Resumo Executivo / Economia Consolidada

> Item 1 do [ROADMAP](../../ROADMAP.md). Status: 🟡 Planejado (ainda não implementado).
> Este documento contém o plano de implementação completo para retomada em
> qualquer sessão futura. Todas as referências de linha são aproximadas
> (`~`) e devem ser reconfirmadas antes de editar, pois `index.html` muda.

## Contexto

O **Corretor Tech** (`index.html`, ~2.600 linhas, SPA de arquivo único) é um gerador de
propostas comerciais para corretores de Bradesco Saúde. Hoje os slides gerados são:
capa → comparativo(s) → coparticipação → carências → diferenciais → rede → contracapa.

O comparativo já calcula, por plano Bradesco, a economia anual vs. o plano atual
(`buildCompSlide`, ~linhas 1459‑1463), mas essa informação aparece dispersa como um
"chip" pequeno embaixo de cada gráfico. **Falta um slide de impacto** que abra a
proposta com o argumento de venda mais forte: *quanto a empresa economiza*, em número
grande e com apoio visual.

**Resultado pretendido:** um novo slide "Resumo da Proposta" (logo após a capa) que
destaca a economia anual consolidada do melhor plano Bradesco, com número herói,
métricas (economia mensal, anual, % de redução), um gráfico de **economia acumulada em
12 meses**, e cartões por plano com a melhor opção em destaque. O slide é opcional
(toggle nas opções de apresentação) e entra automaticamente no PDF.

Combina **nova funcionalidade** (slide + gráfico + toggle) com **ganho visual/UX**
(headline de venda, melhor hierarquia da informação).

## Arquivo a modificar

- `index.html` — único arquivo do projeto. Mudanças em 7 pontos: HTML (opção de slide),
  CSS (estilos do slide) e JS (helper, builder, render, charts, toggle, snapshot).

## Mudanças

### 1. Helper compartilhado de economia (reuso / dedup)
Logo antes de `buildCompSlide` (~linha 1442), criar:

```js
function ganhoInfo(atualValor, planValor) {
  const ganhoMensal = atualValor - planValor;
  const ganhoAnual  = ganhoMensal * 12;
  const cls   = ganhoAnual > 200 ? 'positivo' : ganhoAnual < -200 ? 'negativo' : 'neutro';
  const icon  = ganhoAnual > 200 ? '▼' : ganhoAnual < -200 ? '▲' : '≈';
  const label = ganhoAnual > 200 ? 'Economia' : ganhoAnual < -200 ? 'Invest. adicional' : 'Equivalente';
  const pct   = atualValor > 0 ? (ganhoMensal / atualValor) * 100 : 0;
  return { ganhoMensal, ganhoAnual, cls, icon, label, pct };
}
```
Refatorar as ~linhas 1459‑1463 de `buildCompSlide` para consumir
`ganhoInfo(d.atual.valor, p.valor)` (remove a duplicação do cálculo). Reusa as classes
CSS existentes `.ganho-chip.positivo/negativo/neutro` (definidas ~linhas 175‑178).

### 2. Novo builder `buildResumoSlide(d)`
Adicionar após `buildCover` (~linha 1424). Estrutura reaproveitando padrões existentes:
- Watermark: `const logoWm = d.logoUrl ? '<img ... class="slide-logo-wm">' : ''` (igual aos demais).
- Cabeçalho: bloco `.slide-sec-head` + `.slide-sec-bar` + `.slide-sec-title` (mesmo padrão de
  `buildCopSlide`/`buildCarenciaSlide`), título "Resumo da Proposta".
- Selecionar **melhor plano** = `bradPlans` com `valor>0` de menor valor (maior economia):
  `const melhor = [...d.bradPlans].filter(p=>p.valor>0).sort((a,b)=>a.valor-b.valor)[0]`.
- Calcular `const g = ganhoInfo(d.atual.valor, melhor.valor)`.
- **Banda herói** (flex 2 colunas):
  - Esquerda: número grande `fmt(Math.abs(g.ganhoAnual))` + sufixo "/ano", subtítulo
    economia mensal `fmt(Math.abs(g.ganhoMensal))/mês` e badge "% redução" (`g.pct`).
    Se `g.ganhoAnual <= 0` → reenquadrar para "Investimento adicional" com cor/ícone
    negativos e mensagem de valor agregado (sem número de economia).
  - Direita: `<div class="chart-wrap"><canvas id="ch-resumo-acum"></canvas></div>`
    (gráfico de área de economia acumulada). Só renderiza canvas se `g.ganhoAnual > 0`.
- **Cartões por plano**: um card por `d.bradPlans` mostrando `label`, `fmt(valor)` mensal e
  `ganhoInfo(...).ganhoAnual`/ano; destacar o `melhor` com badge "★ Maior economia"
  (reusa `BRAD_COLORS[i]` para a cor, igual ao comparativo).
- Rodapé: nota "* Valores incluem IOF de 2,38% · estimativa anual" (padrão da ~linha 1522).
- Retornar `<div class="slide-box" id="sl-resumo"><div class="s-resumo">…</div></div>`.

### 3. Inserir no pipeline `render()` (~linha 1360)
Trocar:
```js
let html = buildCover(d) + buildComp(d);
```
por:
```js
let html = buildCover(d);
if (d.showResumo && d.showAtual && d.atual.valor > 0 && d.bradPlans.some(p => p.valor > 0))
  html += buildResumoSlide(d);
html += buildComp(d);
```
(Guarda evita slide sem dados. `gerarPDF` já coleta `.slide-box` por ordem do DOM
[~linha 1956], então o novo slide entra no PDF automaticamente — sem mudança lá.)

### 4. Gráfico de economia acumulada em `drawCharts(d)` (~linha 1828)
No início (antes do `slideGroups.forEach`), se `document.getElementById('ch-resumo-acum')`
existir: criar um Chart.js `type:'line'` com `fill:true`:
- labels = `['M1'..'M12']`, data = economia mensal acumulada (`gMensal * mês`).
- Estilo: `borderColor:'#920031'`, `backgroundColor:'rgba(146,0,49,.12)'`, `tension:.3`,
  `pointRadius:0`, `devicePixelRatio:3`, sem legenda/tooltip, eixo Y com ticks "R$ Xk"
  (reaproveitar o `callback` de `scaleOpts`, ~linha 1880). Fazer `charts.push(...)` para o
  `destroy()` do próximo render (~linha 1356) limpar corretamente.

### 5. Toggle de opção (HTML + `toggleOpt`)
- HTML: na seção "Slides da Apresentação" (~linha 513), adicionar antes de `opt-cop`:
  ```html
  <div class="check-opt" id="opt-resumo-wrap" onclick="toggleOpt('resumo')">
    <input type="checkbox" id="opt-resumo" checked onclick="event.stopPropagation();toggleOpt('resumo')">
    <label for="opt-resumo">📊 Resumo de economia</label>
  </div>
  ```
- `toggleOpt` (~linha 1277): adicionar `resumo: 'showResumo'` ao `map`.
- `state` (init ~linha 1005): adicionar `showResumo: true`.
- `getData()` (return ~linha 1346): adicionar `showResumo: state.showResumo`.

### 6. Persistência (snapshot)
- `getSnapshot` (bloco `state`, ~linha 2427): adicionar `showResumo: state.showResumo`.
- `applySnapshot` (~linha 2481, junto aos outros `showX`): adicionar
  `if (s.showResumo !== undefined) { state.showResumo = s.showResumo; const el=document.getElementById('opt-resumo'); if(el) el.checked=s.showResumo; }`.

### 7. CSS do slide (~linha 91, junto aos demais blocos de slide; referência: `.s-comp` ~158, `.s-cop` ~187)
Adicionar `.s-resumo` (`width:960px; min-height:540px; background:#fff; padding:36px 44px;
position:relative`), grid da banda herói, `.resumo-num` (fonte grande ~52px, peso 800, cor
`--red`), `.resumo-cards` (flex, gap), `.resumo-card` (borda `--border`, radius 10px) e
variação `.resumo-card.best` (borda/realce vermelho). Reusar variáveis `--red`, `--muted`,
`--border`, `--green`.

## Decisões de design
- **Posição:** logo após a capa (headline de venda primeiro, detalhes depois).
- **Plano destacado:** o de menor mensalidade entre os Bradesco (maior economia).
- **Sem economia (Bradesco mais caro):** o slide se adapta para "investimento adicional"
  com enquadramento de valor agregado, em vez de esconder — mantém o slide útil.
- **Múltiplos planos:** herói foca no melhor; cards listam todos para transparência.

## Verificação
1. Abrir `index.html` no navegador (`python3 -m http.server` no diretório e acessar a porta,
   ou abrir o arquivo direto — não depende de backend para renderizar slides).
2. Preencher: empresa, plano atual com valor (ex. 18500), e ≥1 plano Bradesco com valor menor
   (ex. 15200). Conferir que o slide **Resumo da Proposta** aparece como 2º slide com:
   número de economia anual coerente (`(18500-15200)*12 = 39.600`), % de redução, gráfico de
   área crescente, e card do melhor plano com badge "★ Maior economia".
3. Borda: plano Bradesco **mais caro** que o atual → slide mostra "Investimento adicional"
   sem gráfico de economia, sem quebrar.
4. Desmarcar "📊 Resumo de economia" → slide some da prévia e do PDF.
5. Zerar valor do plano atual ou desmarcar "Plano atual" → slide não é renderizado (guarda).
6. Clicar **⬇ Baixar PDF** → confirmar que o resumo entra como página 2, com gráfico nítido.
7. **💾 Salvar** e recarregar pelo histórico → estado do toggle `showResumo` preservado.
8. Conferir comparativo original intacto após a refatoração do `ganhoInfo` (chips de
   economia continuam corretos).
