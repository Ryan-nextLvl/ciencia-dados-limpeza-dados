# Revisão — Limpeza e Preparação de Dados (guia de estudo)

Disciplina: Ciência de Dados · Etapa CRISP-DM: **Preparação dos Dados**

Este guia resume o que cada notebook ensina, com foco em **por que** cada decisão foi tomada, que é o que cai em prova. Os números citados vêm da execução real dos notebooks.

## Como usar para estudar
1. Leia a seção do exercício aqui.
2. Abra o notebook e **tente prever** a saída de cada célula antes de olhar o resultado.
3. Depois, **refaça sem olhar** (copie só o esqueleto do PDF) e compare.
4. Responda a autoavaliação no final em voz alta, sem consultar.

## Datasets (não incluídos no repositório)
Os dados são de terceiros, então não foram versionados. Baixe cada um e coloque na pasta indicada:

| Pasta | Dataset | Onde baixar |
|---|---|---|
| `ex1/` | `melb_data.csv` | https://www.kaggle.com/datasets/dansbecker/melbourne-housing-snapshot |
| `ex2/` | `KPMG_VI_New_raw_data_update_final.xlsx` | https://www.kaggle.com/datasets/archit9406/customer-transaction-dataset |
| `ex3/` | `adult.data` e `adult.names` | https://archive.ics.uci.edu/dataset/2/adult |
| `ex4/` | (nada: `sns.load_dataset('diamonds')`) | carregado pelo seaborn |
| `ex5/` | CSVs da SHEIN | https://www.kaggle.com/datasets/oleksiimartusiuk/e-commerce-data-shein |

Instalação: `pip install -r requirements.txt`

## Estrutura
```
ex1/ exercicio_1.ipynb + melb_data.csv                      (valores ausentes)
ex2/ exercicio_2.ipynb + KPMG_VI_New_raw_data_update_final.xlsx   (duplicatas)
ex3/ exercicio_3.ipynb + adult.data / adult.names           (padronização e escalonamento)
ex4/ exercicio_4.ipynb                                      (outliers; dataset vem do seaborn)
ex5/ exercicio_5.ipynb + 21 CSVs da SHEIN                   (pipeline integrado)
```
Para reexecutar: `python -m jupyter nbconvert --to notebook --execute --inplace exercicio_N.ipynb` dentro da pasta do exercício.
Bibliotecas: pandas, numpy, matplotlib, seaborn, scikit-learn, scipy, missingno, rapidfuzz, openpyxl.

---

## Receita geral de diagnóstico (vale para qualquer dataset)
```python
df.shape                      # dimensões
df.dtypes                     # tipos (número guardado como texto?)
df.isna().sum()               # ausentes "oficiais"
df.duplicated().sum()         # duplicatas exatas
df.describe()                 # min/max revelam valores impossíveis
df[col].value_counts()        # categorias, marcadores como '?' e variações de grafia
```
**Regra de ouro:** explorar antes de limpar, trabalhar em cópias (`df.copy()`), documentar cada decisão com justificativa e impacto, e validar que a limpeza não criou novos problemas.

---

## Exercício 1 — Valores ausentes (Melbourne)
**Conceitos**
- **Remover linhas** perde informação das outras colunas; **remover coluna** perde a variável. **Imputar** preserva o tamanho, mas inventa valores e achata a variância.
- **Medida de imputação:** mediana para numérico assimétrico (robusta a extremos), média só se for simétrico, **moda** para categórico.
- **Padrão de ausência** (missingno): ausentes que aparecem juntas indicam causa comum, ou seja, não são aleatórias.
- **Indicador de ausência** (`coluna_ausente`) guarda o sinal quando faltar é informativo.

**O que os dados mostraram**
- Só 4 colunas com ausentes: `BuildingArea` 47,5%, `YearBuilt` 39,6%, `CouncilArea` 10,1%, `Car` 0,5%.
- `BuildingArea` e `YearBuilt` faltam nas mesmas linhas (correlação de nulidade 0,77).
- Remoção só de linhas: 13.580 → 6.196 (perde 54%). Descartar as 2 colunas pesadas + linhas: 12.211.
- A remoção **não é neutra**: a mediana de `Car` caiu de 2 para 1.
- A imputação mantém a mediana, mas **reduz o desvio** e cria um pico artificial.
- KNNImputer: erro médio 21,4 anos contra 29,1 da mediana, mas é mais caro, exige escalonar, tem risco de vazamento e continua inventando valores.

**Perguntas típicas de prova**
- *Coluna com mais da metade ausente: imputar ou descartar?* Depende da importância da coluna, do motivo da ausência (aleatória ou sistemática), de quanta informação resta e de existirem colunas correlacionadas.
- *Média ou mediana?* Mediana, quando há assimetria (aqui média/mediana = 1,19 em `Price` e 1,27 em `Landsize`).
- *Quando a ausência é informação?* Quando não é aleatória. Exemplo: "renda não informada" em crédito.

---

## Exercício 2 — Duplicatas (KPMG)
**Conceitos**
- **Exata** = linha inteira igual. **Por subconjunto** = mesmas colunas de negócio (ex.: cliente + produto + data), ignorando o id gerado pelo sistema.
- Duplicata exata = 0 **não garante** nada: um reenvio recebe `transaction_id` novo e escapa.
- **Quais colunas definem "o mesmo registro" é decisão de negócio** (Entendimento do Negócio do CRISP-DM), não só técnica.
- **Fuzzy matching** (`rapidfuzz`) sugere candidatos a duplicata por texto (ex.: "Giant Bicycle" x "Giant Bicycles"), mas **não decide**: gera falsos positivos.

**O que os dados mostraram**
- Cabeçalho na 2ª linha: o pandas leu a nota de rodapé como nome de coluna. Correção: `header=1`.
- 0 duplicatas exatas, `transaction_id` único, e 1 par por cliente + produto + data (mas com marcas diferentes, então não é repetição).
- `product_id = 0` aparece em 1.378 linhas, com 6 marcas e 221 preços (parece código-padrão).
- Simulação com 100 reenvios: 0 duplicatas exatas, 100 por subconjunto, soma de `list_price` inflada em 117.120.
- `rapidfuzz` em `job_title` acha 169 pares, mas são níveis diferentes ("Analyst I" x "II"): falsos positivos.

**Perguntas típicas**
- *Duas linhas idênticas são sempre erro?* Não. Ex.: dois itens iguais vendidos no mesmo dia, sem coluna de quantidade ou hora.

---

## Exercício 3 — Padronização e escalonamento (Adult)
**Conceitos**
- `isna()` só vê `NaN`. Marcadores como `?`, `-`, `999` passam batido. É preciso olhar `value_counts` e converter (`replace('?', np.nan)` ou `na_values='?'`).
- Padronizar texto: `strip()`, `lower()`, espaços repetidos. Conferir se há mesmo variação antes de assumir.
- **Min-Max:** `(x − min)/(max − min)`, vai para [0,1], muito sensível a extremos.
- **Z-score:** `(x − média)/desvio`, média 0 e desvio 1, sensível a extremos (média e desvio são puxados por eles).
- **Robusto:** `(x − mediana)/IQR`, resistente a extremos, mas **quebra se IQR = 0**.
- Ajustar (`fit`) o escalador **só no treino**, para não vazar informação do teste.
- Árvores de decisão não precisam de escalonamento. KNN, SVM e redes neurais precisam.

**O que os dados mostraram**
- Todo texto tinha espaço no início (`' Private'`), e `df['workclass'] == 'Private'` retornava **0 linhas** até corrigir. Não havia variação de maiúscula/minúscula.
- `?` escondia 4.262 ausentes (`workclass` 1.836, `occupation` 1.843, `native-country` 583).
- `capital-gain`: 91,7% são zero e 159 linhas têm o teto 99.999. Min-Max joga 94,9% dos valores entre 0 e 0,05. No Z-score o máximo vira 13,4 desvios. No Robusto o IQR real é 0, então nada é escalonado.

**Perguntas típicas**
- *Por que `isna().sum()` dá zero com dados faltando?* Marcadores próprios que o pandas não reconhece.
- *`fnlwgt` deve ser escalonado?* Não. É o **peso amostral** do censo (quantas pessoas a linha representa), não uma característica da pessoa. Usar como `sample_weight` ou descartar.

---

## Exercício 4 — Outliers (Diamonds)
**Conceitos**
- **Z-score:** `|z| > 3`. Supõe distribuição **normal**. Em dados assimétricos, média e desvio são puxados pela cauda.
- **IQR:** fora de `[Q1 − 1,5·IQR, Q3 + 1,5·IQR]`. Baseado em quartis, é mais robusto em dados assimétricos.
- **Erro de registro x extremo legítimo:** o extremo legítimo é **coerente** com o resto da linha; o erro é incoerente por si só.
- **Tratamentos:** remoção (perde linhas), capping (mantém linhas, cria pico no limite), transformação log/raiz (mantém tudo, só muda a escala).
- Para dados assimétricos à direita, `log1p` aproxima de uma distribuição simétrica.

**O que os dados mostraram**
- 20 linhas com dimensão 0, mais `y = 58,9` e `z = 31,8` (o `z` parece ter perdido a vírgula: 3,18).
- Critério físico: `depth ≈ 2·z/(x+y)·100`. Diferença > 5 p.p. ou dimensão ≤ 0 dá **47 erros de registro**. Vários ficam no meio da nuvem de dados e um filtro em `price` não os pegaria.
- IQR marca bem mais que o Z-score em `price` (3.540 x 1.206). A assimetria de `price` é 1,62 e a de `log1p(price)` é 0,12. Fração com |z| > 3: 2,24% (numa normal seria 0,27%).
- **Remover** outliers de `price` apaga 6,5% da base e **87% dos diamantes de 2+ quilates**.

**Perguntas típicas**
- *Qual a suposição do Z-score?* Normalidade. Não vale para `price` (assimétrico).
- *Diamante > 4 quilates: remover?* Depende do objetivo. Para estudar pedras grandes, remover apaga o objeto de estudo.
- *Exemplo em que remover outlier esconde o que o negócio quer?* Detecção de fraude, manutenção preditiva, clientes de alto valor.

---

## Exercício 5 — Pipeline integrado (SHEIN)
**Conceitos**
- Pipeline = **funções pequenas e reutilizáveis**, cada uma `df → df`, encadeadas, com log do impacto de cada etapa.
- **A ordem importa.** Ordem usada: tipos → textos → duplicatas → ausentes → features → outliers. Por quê: duplicatas antes dos ausentes (a escolha da linha mais completa se perderia), duplicatas antes dos outliers (repetições distorcem quartis), features antes do capping (usam o preço real).
- **O significado do vazio muda o tratamento:** `discount` vazio = sem desconto (0), `selling_proposition` vazio = sem selo (0 + indicador), `color-count` vazio = não se aplica (mantém `NA`).
- **Schemas diferentes:** o pipeline tolera colunas faltando. Se a coluna não existe, o resultado é **desconhecido (`NA`)**, nunca 0.
- Outlier de preço: **marcar e criar versão limitada**, sem apagar (preço alto de eletrônico é produto real).

**O que os dados mostraram**
- Preço, desconto e vendas vinham como **texto** (`'$1,199.00'`, `'-14%'`, `'1.5k+ sold recently'`).
- Duplicatas: 358 exatas contra 396 por título + preço (roupas). Em `electronics`, 38 títulos estavam só em outra coluna.
- `rank-sub` misturava `In` e `in`.
- Mesmo pipeline nos 21 arquivos: 21 de 21 processados. Ao fazer isso apareceu o bug de `swimwear` (sem coluna de vendas).
- Negócio: desconto maior **não** implica mais vendas. Spearman +0,14 (roupas) e −0,21 (eletrônicos).
- Ressalva: a ordem das etapas mudou o resultado só de forma modesta aqui (280 → 265 outliers nos eletrônicos). O princípio vale, o efeito depende da base.

---

## Autoavaliação do PDF (respostas curtas)
- **Como diagnosticar logo após carregar?** `shape`, `dtypes`, `isna().sum()`, `duplicated().sum()`, `describe()` e `value_counts()` das categóricas.
- **Remover ou imputar? Qual medida?** Remover quando a coluna é pouco útil ou muito vazia, ou o motivo é sistemático. Imputar quando a coluna importa e há pouca falta. Mediana (numérico assimétrico), média (simétrico), moda (categórico).
- **Duplicata exata x por subconjunto?** Exata = linha inteira igual. Por subconjunto = colunas de negócio iguais, ignorando ids gerados pelo sistema.
- **Dados faltando com `isna()` = 0?** Marcadores próprios (`?`, `-`, `999`) que o pandas lê como valor comum.
- **Número guardado como texto?** `dtypes` mostra `object`/`str`; corrigir com `str.replace(...)` + `pd.to_numeric(errors='coerce')`.
- **Z-score x IQR?** Z-score supõe normalidade e usa média/desvio (afetados pelos extremos). IQR usa quartis e é mais robusto em dados assimétricos.
- **Erro x extremo legítimo?** O legítimo é coerente com o resto da linha; o erro viola regra física ou lógica.
- **Min-Max x Z-score x Robusto?** Intervalo [0,1] (sensível a extremos) x média 0/desvio 1 (sensível) x mediana/IQR (robusto, mas quebra com IQR 0).
- **Por que documentar cada decisão?** Em limpeza raramente há resposta única; o que vale é a decisão estar justificada e o impacto medido, para ser reproduzível e auditável.

## Pegadinhas para lembrar
- `df['col'] == 'Private'` retorna 0 se o texto for `' Private'`: `strip()` primeiro.
- `drop_duplicates` trata `NaN` como igual a `NaN`: cuidado ao deduplicar por colunas com vazios.
- `fillna(0)` só é válido quando vazio **significa** zero.
- Remover linhas com ausentes **muda a distribuição** das outras colunas (a mediana de `Car` mudou).
- Escalonador: `fit` no treino, `transform` no treino e teste.
