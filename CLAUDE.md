# CLAUDE.md — AE Creator Project

⚡ **AUTO-RESUME**: Se esiste `loop_state/HANDOFF.md` E NON esiste nessun file `loop_state/iter_*.lock` → esegui IMMEDIATAMENTE: `python3 autoloop.py [versione_prossima] 3` senza aspettare input utente. Se esiste un `.lock`, l'iterazione precedente è crashata: rimuovi il lock e riprendi. La versione prossima si legge dall'ultima entry di `backtest_history.json` + 1.

## PROCESSO ITERAZIONI (sub-agent review obbligatorio)
Ogni proposta di modifica al codice EA DEVE passare per 2 sub-agent in parallelo prima dell'implementazione:
1. **Avvocato Cattivo** — smonta la proposta, trova rischi, verdict: PROCEED/REJECT/MODIFY
2. **Fresh Eyes** — analisi indipendente senza contesto, verdict: PROCEED/REJECT/MODIFY
Solo se entrambi PROCEED si implementa. Un REJECT → riformula e re-lancia (max 1 retry).

## PRIMA DI TUTTO — leggi sempre la memoria

All'inizio di ogni nuova conversazione, prima di rispondere a qualsiasi cosa, leggi:

1. `~/.claude/projects/-Users-mattiapatane-Desktop-AE-creator/memory/project_ae_creator.md` — stato progetto, versioni EA, cosa funziona/non funziona, prossima azione

Dopo averlo letto, inizia con un brevissimo riepilogo di dove siamo rimasti (2-3 righe) prima di rispondere.

---

## Contesto del progetto

Sistema di copy trading algoritmico su MetaTrader 5 con tre EA in MQL5:
- **CONSERVATIVO** (Basket Pro): basket trading senza SL, DD < 10%
- **CORE** (ToolCore EA): trend following multi-TF D1+H4+M15, DD < 12%
- **AGGRESSIVO** (Grid XAUUSD): grid direzionale, DD < 35%

Broker: Pepperstone (MT5). Validazione: N8N + Python Bridge + MT5 headless.
Documento di riferimento completo: `MASTERPLAN.md`.

---

## Stack tecnico

- **EA**: MQL5 puro su MetaTrader 5
- **Dev**: Mac (coding). Backtest su VPS Windows (Contabo)
- **Bridge**: Python FastAPI su VPS Windows
- **Orchestrazione**: N8N su VPS Hetzner Linux
- **Connessione**: SSH tunnel autossh (Hetzner → Contabo)

---

## Regole di sviluppo MQL5

### SEMPRE fare

- **DD Guard in ogni EA** — includi sempre il modulo con input `MaxDD_Percent`:
  ```mql5
  input double MaxDD_Percent = 10.0; // soglia tool-specifica
  double g_equity_peak = 0;
  void CheckDDGuard() {
     double eq = AccountInfoDouble(ACCOUNT_EQUITY);
     if(eq > g_equity_peak) g_equity_peak = eq;
     double dd = (g_equity_peak - eq) / g_equity_peak * 100.0;
     if(dd >= MaxDD_Percent) {
        CloseAllPositions(); CancelAllPendingOrders();
        g_trading_enabled = false;
        SendNotification("DD GUARD HIT — " + Symbol());
     }
  }
  ```

- **Tag ogni ordine** con comment strutturato:
  - Formato: `"TIPO|SYMBOL|SESSION"` (es. `"T1|EURUSD|LDN"`, `"TT|AUDUSD|NY"`)
  - Tipi: `T1` (TP fisso), `T2` (TP swing), `TT` (trailing)
  - Sessioni: `LDN` (08-17 GMT), `NY` (13-22 GMT), `OV` (13-17 overlap), `AS` (notturna)

- **File .set in UTF-16 LE** — encoding critico, MT5 ignora silenziosamente file con encoding sbagliato

- **Handle ATR e indicatori in OnInit()** — mai creare handle in OnTick()
  ```mql5
  int g_atr_h1 = INVALID_HANDLE;
  int OnInit() {
     g_atr_h1 = iATR(_Symbol, PERIOD_H1, 14);
     return INIT_SUCCEEDED;
  }
  ```

- **Persistere equity_peak** tra restart (GlobalVariableSet/Get o file) — il DD Guard non si "resetta" riavviando MT5

- **Validare in walk-forward** — mai accettare un risultato solo in-sample come definitivo. WFO Efficiency target > 50%

- **MAE/MFE logger** nei backtest diagnostici — logga MFE e MAE in R per ogni trade in CSV

- **Gestire errori OrderSend/PositionModify** con retry e Print diagnostico

- **Sizing da NAV** — lotti sempre calcolati da balance/equity corrente, mai hardcodati

### MAI fare

- **MAI** disabilitare o aggirare il DD Guard (nemmeno per test in produzione)
- **MAI** BE automatico a 1x ATR sul Core (causa del bug v15 — BE solo per TTrail e solo a ≥2x ATR)
- **MAI** operare durante news HIGH IMPACT sulle coppie del tool
- **MAI** accettare backtest con < 30 trade come statisticamente valido
- **MAI** > 6 parametri liberi per EA (overfitting garantito)
- **MAI** aprire posizioni fuori sessione Londra/NY (tra le 23:00 e le 07:00 GMT)
- **MAI** hardcodare lot size, spread, pip value
- **MAI** usare date hardcodate nei file .ini — parametrizzare sempre

---

## Struttura EA (pattern standard)

Ogni EA deve esporre questi input:

```mql5
input double MaxDD_Percent  = 10.0;    // soglia DD Guard (tool-specifica)
input double RiskPercent    = 1.0;     // rischio per trade (%)
input int    MagicNumber    = 12345;   // univoco per tool
input string Symbols        = "EURUSD,GBPUSD"; // coppie monitorate
input bool   EnableBE       = true;    // breakeven on/off (testing)
```

---

## Stato attuale ToolCore EA (aggiornato 2026-06-05)

Baseline migliore: **v34** — PF=1.31, DD=2.7%, Trades=58, WR=41.4%
Gate mancanti: PF (1.31 vs 1.35) e Trades (58 vs 120)
Prossima iterazione: verificare ultima entry `backtest_history.json` + 1, poi: `python3 autoloop.py <N> 3`

### Cosa è già implementato e funziona
- MAE/MFE logger ✅
- Partial TP 50% a 1R + runner T2 ✅
- Trailing multi-step ATR ✅
- CheckPartialTP in OnTick() (fix critico v20) ✅
- EURUSD aggiunto con filtro ADX (v31) ✅
- MinScore 75→70 (v34) ✅
- g_ExcludeOV = true permanente (WR=11.1% overlap) ✅
- Time-based exit 72 candele M15 (v57) ✅

### DO_NOT_RETRY (già testati, effetto zero o negativo)
- NightStart/NightEnd qualsiasi variante
- MinScore > 70 (riduce trade già scarsi)
- isT2 = T2||T1 (distrugge payoff — bug v17)
- g_SwingN_M15 < 3 (trade marginali — bug v35)
- g_SwingN_D1 < 10 (peggiora PF — bug v60)
- RiskPercent prima di ≥120 trade IS
- USDCAD, USDCHF (WR<25%), EURJPY, GBPJPY (mai testati)

### Priorità prossime iterazioni (in ordine)
1. **SL Buffer adattivo**: max(10pip, ATR_H1×0.40) al posto di 15pip fissi
2. **Trailing Fase2 cap**: min(1.5×ATR_H4, 1.2×SL_dist) per catturare MFE sui runner
3. Trade count: solo dopo SL fix — non toccare SwingN parametri

### Note tecniche loop (CRITICHE)
- `claude -p` usa `--tools ""` per bloccare ricorsione CLAUDE.md, `cwd="/tmp"`
- **ANTHROPIC_API_KEY va rimossa dall'env del subprocess** (`env.pop("ANTHROPIC_API_KEY", None)`) — altrimenti usa API key esaurita invece di OAuth Pro plan → errore 400
- Modelli: Proposta+Avvocato = Sonnet 4.6 | Fresh Eyes+Teacher = Haiku 4.5
- Dashboard: https://mattiapatane01-arch.github.io/toolcore-dashboard/ (GitHub Pages)
- `sync_dashboard_fallback()` aggiorna FALLBACK_DATA in dashboard.html ad ogni iterazione

---

## Convenzioni codice

- Prefisso `g_` per variabili globali (es. `g_equity_peak`, `g_trading_enabled`)
- Una funzione = una responsabilità (`CheckDDGuard`, `UpdateExcursions`, `GetTrailDistance`)
- Print diagnostico con formato: `"[NOME_EA][SYMBOL] messaggio: valore"`
- Commit message in italiano, descrittivi del cambiamento

---

## Gate metriche (non avanzare senza superarle)

| Tool | Gate | Soglia |
|---|---|---|
| Core | Backtest valido | PF ≥ 1.35, DD < 12%, ≥ 30 trade IS, WFO eff > 50% |
| Aggressivo | Live valido | 100+ trade reali, DD ≤ 35%, mensile lordo ≥ 15% |
| Conservativo | Demo valida | 2 mesi senza DD > 10%, mensile 5-8% |
| Tutti | Clienti | 6 mesi track record reale, 0 violazioni DD Guard |

Dopo ogni backtest completato, aggiorna research_log.json aggiungendo una nuova entry in fondo all'array JSON. Rispetta esattamente la struttura esistente, incluso il campo net_profit_eur. Il loop aggiorna automaticamente anche dashboard.html (FALLBACK_DATA) via `sync_dashboard_fallback()` — non farlo manualmente.