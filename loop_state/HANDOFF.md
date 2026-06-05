# HANDOFF ToolCore Loop
**Versione**: v60 testata → crea v61
**Status**: BACKTEST_OK
Leggi `backtest_history.json` ultima entry per metriche complete, diff e reasoning.

## Metriche v60
PF=1.01 | DD=3.89% | Payoff=2.233913043478261 | Trades=58 | WR=31.03%
Gate: pf ✗ | dd ✓ | payoff ✓ | trades ✗

## Problema aperto
v60: PF=1.01 — vedere LEARNINGS.md per diagnosi

## Prossima ipotesi
Da determinare nella prossima iterazione basandosi sui nuovi dati MAE/MFE

## DO_NOT_RETRY (ultime versioni)
- MinScore 60→72: non affronta causa radice
- isT2 = T2||T1: distrugge payoff (bug v17)
