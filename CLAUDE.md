# Simuladores Renda Fixa

Ferramenta web de simulação de renda fixa para assessoria de investimentos (mercado brasileiro).
Publicada no GitHub Pages: https://jonathancaye.github.io/simuladores/

Nasceu da conversão de planilhas Excel do usuário (`Simulador_Bancarios_aai.xlsm`, `GrossUp_1.xlsm`,
`NEW_GROSSUP.xlsx`) em uma página dinâmica. Tudo em **português (pt-BR)**, números no formato brasileiro.

## Como trabalhar neste projeto

- **Ajustes pontuais por rodada.** O usuário prefere mudanças direcionadas e aprova antes de mudanças
  grandes. Para algo estrutural (nova aba, mudança de cálculo, reorganização de tela), proponha primeiro
  e espere o "pode seguir".
- **Teste antes de enviar** (ver "Como testar") e confira o visual no desktop e no celular (~400 px).
- Commits vão direto para `main` (o Pages publica a partir de `main`, pasta raiz). O site atualiza ~1 min após o push.
- Ao terminar, explique em poucas linhas o que mudou na tela, com um exemplo numérico quando ajudar.

## Arquivos

| Arquivo | O que é |
|---|---|
| `index.html` | A aplicação inteira: HTML, CSS e JS em um único arquivo, sem dependências além de Google Fonts. **É o código-fonte**, edite-o diretamente. |
| `indicadores.json` | Indicadores oficiais, gerado pela Action. Não editar à mão. |
| `scripts/atualiza_indicadores.py` | Busca os indicadores no Banco Central e grava `indicadores.json`. |
| `.github/workflows/indicadores.yml` | Action agendada (dias úteis 09:20 e 12:20 BRT) + `workflow_dispatch`. Só faz commit se algo mudou. |

## Abas (ordem e atalhos de teclado 1–6)

1. **Simulador** (`#simulador`) — cálculo **estático**: um produto, CDI e IPCA constantes. Ficha da aplicação,
   resultado (valor líquido, rentab. no período, taxa líquida a.m./a.a. base 252), demonstrativo, histórico
   (localStorage) e "Ver gross-up".
2. **Comparador** (`#comparador`) — cálculo **dinâmico por cenários**. Escolha de quais opções comparar
   (Pré, %CDI, CDI+, IPCA+), isenção por coluna. Cenários **Queda / Neutro / Alta**: neutro = Focus anual
   (editável célula a célula), queda/alta = neutro ∓ choque (padrão 2 p.p. CDI, 1 p.p. IPCA).
   Cabeçalho verde: "Melhor no cenário neutro" + linha de **menor arrependimento** (opção que menos perde,
   no pior caso, para a vencedora de cada cenário; o antigo critério "melhor no pior cenário" favorecia o Pré
   porque a queda derruba CDI e IPCA juntos) e cards com a melhor opção de cada cenário + CDI/IPCA médios.
   Tabela unificada, amplitude, diferença para a melhor do neutro, maior perda (arrependimento), detalhes do
   neutro (recolhível), gráfico de faixa (Q/N/A, linha da melhor do neutro, ★ na vencedora de cada cenário),
   pontos de empate **contra uma referência** (padrão: melhor do neutro; pills para trocar, `C.ref`), com o
   outro indicador fixo no neutro (`pairBE`), premissas ano a ano (recolhível). "Abrir no simulador" leva o CDI/IPCA médios
   do neutro; "Levar para a carteira".
3. **Carteira** (`#carteira`) — posições editáveis (produto, IR, taxa, vencimento, valor em R$ ou %),
   gerador de **escada (ladder)** com inclinação de taxa por degrau, indicadores, cronograma de vencimentos
   (barras principal × juros), composição, e **reinvestimento da escada** (horizonte, taxa: degrau longo /
   informada / ajuste; reaplicar tudo ou resgatar juros; comparação com aplicação única).
4. **Capital protegido** (`#protegido`) — quanto aplicar na renda fixa para recompor o capital (nominal ou
   corrigido pelo IPCA) no vencimento; saldo para risco, cenários do ativo de risco, ponto de empate com 100% RF.
   O retorno do ativo de risco é tratado como **já líquido** de impostos.
5. **Gross-up** (`#grossup`) — isento ↔ tributável nos 4 indexadores, equivalência por indexador,
   resultado em R$, curva da taxa equivalente por prazo (composto × aproximação linear).
6. **Equivalência de taxas** (`#equivalencia`) — converte uma taxa entre Pré, %CDI, CDI+ e IPCA+;
   sensibilidade ao CDI; alterna %CDI diário (mercado) × anual simples (planilha).

## Estrutura do código (`index.html`)

- O `<script>` é um IIFE. Seções marcadas com `/* =================== MÓDULO: ... */`
  (o Gross-up vem primeiro, sem marcador; depois Simulador, Equivalência, Comparador, Carteira,
  Reinvestimento, Capital protegido, Abas, Indicadores).
- Prefixo dos ids por aba: `s` Simulador · `c`/`sc` Comparador e cenários · `p`/`l`/`r` Carteira, escada e
  reinvestimento · `k` Capital protegido · `e` Equivalência · Gross-up sem prefixo.
- Estado por aba em objetos (`S` gross-up + projeções globais `S.cdi`, `S.ipca`, `S.manual`; `T` simulador;
  `C` comparador; `P` carteira; `K` capital protegido; `E` equivalência), salvos em `localStorage`
  (chaves `*.v1`). Ao adicionar campos, mescle com os defaults para não quebrar dados salvos.
- `syncAll()` re-renderiza todas as abas. Cada aba tem seu `xxxRender()`.
- Funções-base reutilizadas: `cumBiz`/`isBiz` (dias úteis), `irRate`, `iofRate`, `toAnnual`/`fromAnnual`,
  `simCalc` (resultado com CDI/IPCA constantes), `scenCalc` (resultado numa trajetória de CDI/IPCA),
  `breakeven`, `fmtRate`, `nf`, `brl`, `parseNum`, `copyText`, `niceTicks`.
- Feriados nacionais 2020–2078 embutidos na constante `HOL` (da planilha), + 20/11 a partir de 2024.
- Cuidado com nomes de classe genéricos já usados (`.sep`, `.best`, `.win`, `.total`): crie nomes novos
  para evitar conflito de CSS.

## Convenções de cálculo (não mudar sem combinar)

- Capitalização em **dias úteis, base 252**. Dias úteis = da aplicação até a véspera do vencimento.
- **IR regressivo pelos dias corridos**: ≤180 22,5% · ≤360 20% · ≤720 17,5% · >720 15%.
  (A planilha original somava +1 dia — corrigido.) **IOF** regressivo abaixo de 30 dias (tributados).
- **%CDI pela convenção de mercado** (fator diário `1 + %·((1+CDI)^(1/252)−1)`); a planilha usava CDI × %.
  A aba Equivalência permite ver as duas.
- CDI+ e IPCA+: composição `(1+índice)(1+spread)−1`.
- Gross-up composto pelo prazo (a planilha usava aproximação linear taxa × (1−IR)).
- Taxas líquidas a.m./a.a. exibidas em **base 252** (a.m. = 21 dias úteis).
- Cenários: CDI de fim de ano = Selic Focus − 0,10 p.p.; trajetória interpolada linearmente a partir do
  CDI atual; IPCA aplicado ano a ano; "CDI/IPCA médio" = média composta anualizada do período.
- Não usar a palavra "garantida" para rentabilidade (confunde com garantia de recebimento):
  usar "rentabilidade definida no momento da aplicação".

## Indicadores (`indicadores.json`)

Campos: `cdi` (SGS 4389), `ipca_12m` (SGS 13522), `selic_focus` (Focus, fim do ano corrente),
`ipca_focus_12m` (Focus 12 meses, suavizada), `focus_anual` (`selic` e `ipca` por ano, ano corrente + 4),
`atualizado_em`, e `erros` quando alguma consulta falha (o valor anterior é mantido).

- A API Olinda do BCB bloqueia (403) alguns filtros combinados; por isso o script tenta um filtro só por
  `Indicador` e filtra o resto em Python. Os parâmetros `$filter` etc. **não** podem ter o `$` codificado.
- Na página: CDI projetado = `cdi`; IPCA projetado = `ipca_focus_12m`. Se o usuário digitar outro valor,
  a página entra em "Projeção manual" (botão "usar oficiais" volta).
- A linha de status do topo mostra CDI, IPCA 12M, Selic Focus e IPCA Focus 12M.
- O ambiente do Claude normalmente **não** acessa o BCB; para testar o script, dispare a Action
  (`POST /repos/jonathancaye/simuladores/actions/workflows/indicadores.yml/dispatches`) e leia o
  `indicadores.json` atualizado no repositório.

## Visual

- Estilo **terminal financeiro, fundo branco** (sem modo escuro), compacto, cantos retos, rótulos em
  IBM Plex Mono maiúsculo, texto em IBM Plex Sans Condensed.
- Acentos **verde-petróleo** (`--teal #009b97`, `--teal-ink`) e **laranja** (`--orange #d9701a`).
- **Painel de parâmetros** à esquerda com fundo azul-acinzentado (`--side`), resultados à direita em branco.
- Cada bloco tem barra de título (`.ph` / `.card-head`). Destaques de "melhor opção" em faixa verde.
- Pós-fixados sempre com aviso de que o resultado é projetado e pode variar com os indicadores.
- Rodapé: não constitui recomendação de investimento.

## Como testar

O ambiente bloqueia Google Fonts e o BCB; isso não afeta o teste. Sirva a pasta por HTTP (o
`indicadores.json` só carrega via HTTP) e use o Playwright (Chromium já instalado):

```bash
cd /caminho/do/repo && python3 -m http.server 8766 &
# abrir http://localhost:8766/index.html#comparador, checar erros de console (pageerror),
# conferir números-chave e tirar screenshot em 1366 px e 400 px
```

Valores de referência úteis: Pré 14,65% de 10/12/2024 a 10/12/2025, R$ 100 mil → bruto R$ 14.650,00,
IR 17,5% R$ 2.563,75, líquido R$ 12.086,25 (bate com a planilha original).
