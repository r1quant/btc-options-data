
```python
--- Report ---
  Snapshots: 807
  Total rows: 819100
  Date: 2026-01-23 01:00:00 - 2026-03-05 19:00:00
  Columns: ['date', 'instrument_name', 'expiry_name', 'expiry_date', 'strike', 'type', 'volume', 'open_interest', 'rho', 'theta', 'vega', 'gamma', 'delta', 'bid_iv', 'ask_iv', 'mark_iv']
  CSV: full_data.csv (116.1 MB)
  PARQUET: full_data.parquet (17.3 MB)
  Samples: full_data_sample.csv, sample.csv
```

## BTCUSD_GEX_FEATURES.csv

Os [cálculos](https://www.youtube.com/watch?v=kAfw2Mztf8o) foram desenvolvidos pelo [Leandro Guerra](https://github.com/leandrowar)

Dataset combinando dados OHLC horários do BTC/USD com features extraídas da cadeia de opções (GEX, PCR, IV skew, etc.).

### Como ler

```python
import pandas as pd

df = pd.read_csv("btcusd_gex_features.csv", parse_dates=["date"])
print(df.shape)   # (1003, 671)
print(df.head())
```

### Estrutura das colunas

O arquivo contém ~671 colunas divididas em 3 grupos:

| Grupo | Prefixo | Quantidade | Descrição |
|-------|---------|-----------|-----------|
| OHLC | — | 7 | Dados de preço horários |
| Global GEX | `gex_` | 3 | Metadados do snapshot |
| Per-expiry | `exp{N}_` | ~661 | Features por vencimento (ordenados por DTE) |

---

### 1. Colunas OHLC

| Coluna | Descrição |
|--------|-----------|
| `date` | Timestamp da barra horária |
| `open` | Preço de abertura |
| `high` | Preço máximo |
| `low` | Preço mínimo |
| `close` | Preço de fechamento |
| `volume` | Volume em USD |
| `turnover` | Volume em BTC |

### 2. Colunas globais (`gex_`)

| Coluna | Descrição |
|--------|-----------|
| `gex_spot` | Preço spot do BTC no momento do snapshot |
| `gex_timestamp` | Timestamp da análise |
| `gex_num_expiries` | Quantidade de vencimentos analisados naquele snapshot |

### 3. Colunas per-expiry (`exp{N}_`)

Cada vencimento recebe um índice `N` (0, 1, 2, ...) ordenado por **DTE** (days to expiry) — `exp0` é o vencimento mais próximo, `exp1` o segundo, etc. Todas as features abaixo são repetidas para cada `N`.

#### 3.1 Identidade do vencimento

| Sufixo | Exemplo | Descrição |
|--------|---------|-----------|
| `_expiry_name` | `exp0_expiry_name` | Nome do vencimento (ex: `6MAR26`) |
| `_dte` | `exp0_dte` | Dias até o vencimento |
| `_expiry_type` | `exp0_expiry_type` | Classificação: `daily`, `weekly`, `monthly`, `quarterly` |
| `_is_friday` | `exp0_is_friday` | Se o vencimento cai numa sexta-feira |
| `_is_monthly` | `exp0_is_monthly` | Se é vencimento mensal ou trimestral |

#### 3.2 Key Levels

| Sufixo | Descrição |
|--------|-----------|
| `_key_levels_gamma_flip` | Strike onde o GEX cruza zero (dealers passam de long gamma para short gamma) |
| `_key_levels_gamma_flip_pct_distance` | Distância % do spot até o gamma flip |
| `_key_levels_spot_above_gamma_flip` | `True` se spot está acima do gamma flip (dealers short gamma → mais volatilidade) |
| `_key_levels_call_wall` | Strike com maior GEX positivo (resistência por gamma de calls) |
| `_key_levels_call_wall_distance_pct` | Distância % do spot até a call wall |
| `_key_levels_put_wall` | Strike com maior GEX negativo absoluto (suporte por gamma de puts) |
| `_key_levels_put_wall_distance_pct` | Distância % do spot até a put wall |
| `_key_levels_call_put_wall_same` | `True` se call wall e put wall estão no mesmo strike |
| `_key_levels_pcr` | Put/Call Ratio (OI de puts / OI de calls) |
| `_key_levels_pcr_regime` | Sentimento derivado do PCR: `Bullish`, `Bearish`, `Neutral` |
| `_key_levels_iv_skew_pct` | IV skew = IV média de puts OTM - IV média de calls OTM. Valores altos indicam hedge de downside |

#### 3.3 Market Regime

| Sufixo | Descrição |
|--------|-----------|
| `_regime_volatility` | Regime de volatilidade: `HIGH`, `LOW`, `NORMAL`, `UNKNOWN` |
| `_regime_dealer_gamma` | Posição gamma dos dealers: `long_gamma` (spot < gamma flip, mercado estável) ou `short_gamma` (spot > gamma flip, mais volátil) |

#### 3.4 GEX Zones (Suporte/Resistência)

| Sufixo | Descrição |
|--------|-----------|
| `_gex_top1_resistance_level` | Strike da maior zona de resistência GEX |
| `_gex_top1_resistance_value` | Valor GEX da zona (em milhões USD) |
| `_gex_top1_resistance_strength` | Força da resistência |
| `_gex_top2_resistance_level` | Strike da 2a maior zona de resistência |
| `_gex_top2_resistance_value` | Valor GEX da 2a zona |
| `_gex_top1_support_level` | Strike da maior zona de suporte GEX |
| `_gex_top1_support_value` | Valor GEX da zona de suporte (em milhões USD) |
| `_gex_top1_support_strength` | Força do suporte |
| `_gex_top2_support_level` | Strike da 2a maior zona de suporte |
| `_gex_top2_support_value` | Valor GEX da 2a zona de suporte |
| `_gex_total_resistance` | Soma total do GEX positivo (resistência) em milhões |
| `_gex_total_support` | Soma total do GEX negativo (suporte) em milhões |
| `_gex_net_balance` | Balanço líquido GEX (resistance + support). Positivo = mercado mais "travado" |
| `_gex_resistance_count_strong` | Quantidade de strikes acima do spot com GEX > 200M |
| `_gex_support_count_strong` | Quantidade de strikes abaixo do spot com GEX > 200M |

#### 3.5 PCR por faixa de strike

A cadeia de opções é dividida em faixas percentuais ao redor do spot. Cada faixa tem seu próprio Put/Call Ratio.

| Sufixo | Faixa | Descrição |
|--------|-------|-----------|
| `_pcr_range_below95_pcr` | < 95% do spot | PCR de opções deep OTM puts — mede hedge de cauda |
| `_pcr_range_96_98_pcr` | 96-98% do spot | PCR de puts levemente OTM |
| `_pcr_range_98_100_pcr` | 98-100% do spot | PCR de opções near-the-money (lado put) |
| `_pcr_range_atm_pcr` | ~100% do spot | PCR at-the-money |
| `_pcr_range_101_102_pcr` | 101-102% do spot | PCR de calls levemente OTM |
| `_pcr_range_102_104_pcr` | 102-104% do spot | PCR de calls OTM |
| `_pcr_skew_otm_vs_atm` | — | Diferença entre PCR OTM puts (below95) e PCR ATM. Alto = forte posicionamento defensivo |

#### 3.6 Pin Candidates

Strikes onde o preço tende a ser "magnetizado" (pin) no vencimento, baseado na concentração de OI e GEX.

| Sufixo | Descrição |
|--------|-----------|
| `_pin_top1_level` | Strike do pin candidate #1 |
| `_pin_top1_gex` | GEX do dealer nesse strike |
| `_pin_top1_distance_pct` | Distância % do spot ao pin |
| `_pin_top1_pcr` | Put/Call ratio nesse strike específico |
| `_pin_top2_level` | Strike do pin candidate #2 |
| `_pin_top2_gex` | GEX do dealer no pin #2 |
| `_pin_top2_distance_pct` | Distância % do spot ao pin #2 |
| `_pin_top2_pcr` | Put/Call ratio no pin #2 |
| `_nearest_pin_distance_pct` | Distância % do spot ao pin mais próximo |
| `_pin_count_total` | Quantidade total de pin candidates |

#### 3.7 Top Open Interest

Os strikes com maior open interest para calls e puts.

| Sufixo | Descrição |
|--------|-----------|
| `_oi_call_top1_strike` | Strike da call com maior OI |
| `_oi_call_top1_oi` | Open interest dessa call |
| `_oi_call_top1_iv` | IV média (bid+ask/2) dessa call |
| `_oi_call_top2_strike` | Strike da 2a maior call OI |
| `_oi_call_top2_oi` | OI da 2a call |
| `_oi_call_top2_iv` | IV da 2a call |
| `_oi_put_top1_strike` | Strike da put com maior OI |
| `_oi_put_top1_oi` | Open interest dessa put |
| `_oi_put_top1_iv` | IV média dessa put |
| `_oi_put_top1_distance_pct` | Distância % do spot ao top put strike (mede quão longe está o hedge) |
| `_oi_put_top2_strike` | Strike da 2a maior put OI |
| `_oi_put_top2_oi` | OI da 2a put |
| `_oi_put_top2_iv` | IV da 2a put |
| `_oi_put_call_oi_ratio_top5` | Ratio OI top5 puts / top5 calls |

#### 3.8 Derived / Cross-section

| Sufixo | Descrição |
|--------|-----------|
| `_spot_vs_call_wall_pct` | Distância % spot → call wall |
| `_spot_vs_put_wall_pct` | Distância % spot → put wall |
| `_spot_vs_gamma_flip_pct` | Distância % spot → gamma flip |
| `_gex_regime_x_pcr` | Interação regime volatilidade + sentimento PCR (ex: `HIGH_Bearish`) |
| `_iv_skew_x_pcr` | Produto IV skew * PCR — captura correlação entre skew e posicionamento direcional |

### 4. Colunas cross-expiry (`exp{I}_exp{J}_`)

Comparam features entre vencimentos consecutivos (ex: `exp0_exp1_` compara o 1o e 2o vencimento mais próximos).

| Sufixo | Descrição |
|--------|-----------|
| `_gamma_flip_diff` | Diferença no gamma flip entre vencimentos — indica se a estrutura a termo muda |
| `_pcr_diff` | Diferença no PCR entre vencimentos |
| `_call_wall_diff` | Diferença na call wall entre vencimentos |
| `_iv_skew_diff` | Diferença no IV skew entre vencimentos |
| `_oi_put_ratio` | Ratio do OI de puts do vencimento curto / vencimento longo |

### Glossário

| Termo | Significado |
|-------|-------------|
| **GEX** | Gamma Exposure — mede quanto os dealers precisam comprar/vender o subjacente para fazer delta hedge. GEX positivo = resistência (dealers vendem rallies), GEX negativo = suporte (dealers compram dips) |
| **Gamma Flip** | Strike onde GEX cruza zero. Acima dele, dealers estão short gamma (amplificam movimentos); abaixo, long gamma (amortecam movimentos) |
| **Call Wall** | Strike com maior GEX positivo — age como teto/resistência |
| **Put Wall** | Strike com maior GEX negativo absoluto — age como piso/suporte |
| **PCR** | Put/Call Ratio — OI de puts / OI de calls. >1.1 = bearish, <0.9 = bullish |
| **IV Skew** | Diferença de IV entre puts OTM e calls OTM. Alto = demanda por proteção de queda |
| **Pin** | Strike com alta concentração de OI que "atrai" o preço no vencimento |
| **DTE** | Days to Expiry — dias até o vencimento da opção |
| **OTM** | Out of The Money — opção fora do dinheiro |


## Reading Parquet files with Pandas and Polars

### Pandas

```python
import pandas as pd

# Basic reading
df = pd.read_parquet("file.parquet")

df = pd.read_parquet("file.parquet", engine="pyarrow")

print(df.head())
print(df.dtypes)
```

---

### Polars

```python
import polars as pl

# Basic reading
df = pl.read_parquet("file.parquet")

# Reading only the main columns
df = pl.read_parquet("file.parquet")

print(df.head())
print(df.schema)
```

---

### Examples

#### Lazy reading with Polars (recommended for large files)

```python
import polars as pl

# Filtering only CALL options with volume > 0, for a specific instrument
df = (
    pl.scan_parquet("file.parquet")
      .filter(
          (pl.col("type") == "C") &           # only CALL options
          (pl.col("volume") > 0) &            # with volume
          (pl.col("date") >= "2024-01-01")    # from a specific date
      )
      .select([
          "date",
          "instrument_name",
          "expiry_date",
          "strike",
          "type",
          "volume",
          "open_interest",
          "delta",
          "mark_iv"
      ])
      .sort("date")
      .collect()
)
```

#### Practical examples using the Greeks 

```python
import polars as pl

df = pl.scan_parquet("file.parquet")

# Options with high gamma and significant volume
high_gamma = (
    df.filter(
        (pl.col("gamma") > 0.05) &
        (pl.col("volume") > 100)
    )
    .select(["date", "instrument_name", "strike", "type", "gamma", "delta", "volume"])
    .collect()
)

# IV spread (ask_iv - bid_iv) per instrument
iv_spread = (
    df.select([
        "date",
        "instrument_name",
        "strike",
        "bid_iv",
        "ask_iv",
        (pl.col("ask_iv") - pl.col("bid_iv")).alias("iv_spread")
    ])
    .sort("iv_spread", descending=True)
    .collect()
)
```

The lazy API is especially useful here since options data can get very large — filtering by `date`, `type`, or `volume` before loading everything into memory makes a big difference in performance.