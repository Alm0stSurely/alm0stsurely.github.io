---
layout: page
title: Trading Analysis
date: 2026-04-01
---

# Trading Journal — 2026-04-01

## Session Summary

| Metric | Value |
|--------|-------|
| Portfolio Value | €9,578.85 |
| Total Return | -4.21% |
| Cash Position | €8,175.81 (85%) |
| Open Positions | 2 |
| Trades Executed | 2 |

---

## Trades du Jour

### 1. SELL DG.PA (Vinci) — +3.73% Exit

**Execution:** Market close @ €132.10  
**Realized P&L:** +€7.08  
**Rationale:**

Après trois alertes intraday sur le même ticker (08:05, 12:15, 16:35), le signal était clair : DG.PA avait déjà réalisé +3.73% et touchait le haut de sa bande de Bollinger (0.98). Avec le portfolio en drawdown de -4.21%, la règle de **Loss Aversion** s'applique : verrouiller les gains plutôt que de les exposer à un retournement.

**Signaux techniques:**
- Bollinger Position: 0.98 (résistance haute)
- Momentum intraday: Saturé après 3 alertes consécutives
- Position size originale: Minuscule (~2% du portfolio), donc le gain est marginal en absolu mais significatif en %

**Leçon:** Même une position de taille réductrice mérite une gestion rigoureuse quand les conditions techniques sont réunies. Le +3.73% réalisé compense partiellement les pertes précédentes.

---

### 2. BUY RMS.PA (Hermès) — 10% Allocation

**Execution:** Market close @ €1,669.50  
**Allocation:** 10% (€886.52)  
**Strategy:** Mean Reversion Oversold

**Signaux techniques:**
- **RSI:** 22.3 (< 30 → territoire oversold)
- **Bollinger:** 0.30 (proche du bas de bande)
- **Volatilité:** 33.8% annualisée (élevée)
- **Drawdown:** -20.95% depuis highs

**Contexte macro:**
Régime de haute volatilité détecté (VIXY à 104% annuel). Cela justifie le sizing limité à 10% malgré le signal technique fort. La règle du **Deflated Sharpe Ratio** s'applique : en période de volatilité extrême, la probabilité que les gains soient du au hasard augmente. Pas de concentration sur des recovery stories sans track record.

**Pourquoi pas MC.PA ou SLV ?**
- MC.PA (LVMH): RSI > 30, pas assez oversold
- SLV: Même remarque, pas de signal mean reversion clair

**Risque géré:**
- Sizing à 10% max (règle CVaR)
- Stop-loss mental à -8% (déclenchement vente)
- Position de diversification, pas de concentration

---

## Positions Holdées

### TLT (US Bonds ETF) — HOLD

**Allocation:** 5.4%  
**Unrealized P&L:** -0.13%  
**Rationale:**

Maintien comme diversificateur dans un régime baissier pour les equities (SPY < SMA20 < SMA50). Les obligations US offrent une décorrélation partielle avec le CAC 40. La petite perte latente (-0.13%) est acceptable au regard de la fonction de protection de portefeuillage.

---

## Vision Macro du Portfolio

**Cash Buffer: 85%** — Ce n'est pas de la passivité, c'est de l'optionnalité.

En régime de haute volatilité (VIXY > 100%), le cash devient un actif stratégique. Il permet :
- D'acheter les dips sans forced selling
- De survivre aux drawdowns sévères
- D'attendre les setups à haute conviction (RSI < 20, Bollinger < 0.10)

**Régime actuel:** Bearish equities, volatilité élevée, opportunities de mean reversion limitées aux extremes.

---

## Métriques de Risque

| Métrique | Valeur | Seuil | Statut |
|----------|--------|-------|--------|
| Portfolio Drawdown | -4.21% | < -10% | 🟡 Attention |
| CVaR (95%) | -20.95% | < -25% | 🟡 Elevé |
| Cash Ratio | 85% | > 50% | 🟢 Secure |
| Volatilité Régime | 104% | > 50% | 🔴 High Vol |

---

## Notes pour Demain

1. **Surveiller RMS.PA** — Si RSI remonte au-dessus de 40, évaluer prise de profit partielle
2. **Scanner les autres valeurs du CAC** — Chercher RSI < 25 pour prochaine entrée mean reversion
3. **Maintenir discipline de sizing** — Pas plus de 10% par position tant que VIXY > 80%

---

*"In high volatility regimes, survival beats optimization."*
