# HANDOFF ToolCore Loop
**Versione corrente**: v29 → prossima v30
**Data**: 2026-06-04
**Status**: IN_PROGRESS — gate trade non superato

## Stato attuale
| Ver | PF | DD% | Trades | Note |
|-----|-----|-----|--------|------|
| v25 | 1.22 | 15.9% | 96 | Migliore volume trade |
| v29 | 1.35 | 4.0% | 29 | Migliore PF/DD ma troppo pochi trade |

## ⚠️ Problema principale
v29 ha PF=1.35 e DD=4% ma solo 29 trade in 4 anni = 7/anno.
NON è un prodotto vendibile. Serve ≥120 trade IS (≥30/anno).
Il sistema opera quasi non opera: troppi simboli rimossi + RiskPercent troppo basso.

## Gate da superare TUTTI
- PF ≥ 1.35 ✓ (v29 ce l'ha)
- DD ≤ 12% ✓ (v29 ce l'ha)
- Trades ≥ 120 su IS 2020-2023 ✗ (v29 ha solo 29)
- Frequenza ≥ 30/anno ✗ (v29 ha 7/anno)

## DO_NOT_RETRY
- NightStart/NightEnd (v21-v23, zero effetto)
- MinScore ↑ oltre 60 (riduce trade già scarsi)
- Rimuovere altri simboli (già troppo pochi: AUDUSD, NZDUSD, USDJPY)
- Ridurre ulteriormente RiskPercent (già 0.34%)

## STEP DA FARE DOMANI (in ordine)
### Step 1 — Ripristina simboli e frequenza
Riaggiungi EURUSD e GBPUSD alla lista simboli.
Sono ad alta liquidità, spread basso, ottima correlazione con il sistema trend-following.
old_code: `input string g_SymbolsCSV        = "AUDUSD,NZDUSD,USDJPY"; // Simboli monitorati`
new_code: `input string g_SymbolsCSV        = "EURUSD,GBPUSD,AUDUSD,NZDUSD,USDJPY"; // Simboli monitorati`

### Step 2 — Alza RiskPercent a livello sostenibile
Con più simboli e DD sotto controllo, RiskPercent 0.34% è troppo conservativo.
Alzalo a 0.5% per avere sizing ragionevole senza rischiare il DD gate.

### Step 3 — Analizza MAE/MFE della nuova versione
Con più trade e più simboli, rianalizza quali coppie/sessioni contribuiscono negativamente.
Solo allora decidi eventuali ulteriori filtri.

### Step 4 — Walk-forward su v_ottimale
Quando hai una versione con ≥120 trade e PF≥1.35:
python3 best_versions/backtest_extra.py --version N --period oos
Verifica che i risultati tengano su 2024 (dati mai visti).

## Comando per riprendere
```bash
cd "/Users/mattiapatane/Desktop/AE creator" && python3 autoloop.py 30 3
```

## Note tecniche importanti
- .set file rimosso prima di ogni backtest ✓ (fix critico del 2026-06-04)
- make_ini() legge parametri dal sorgente ✓
- Teacher rileva risultati identici ✓
- GitHub Pages: https://mattiapatane01-arch.github.io/toolcore-dashboard/
- Netlify esauriti fino al 29 giugno — NON usare
