# Trading Analysis — 2026-03-13

**Session:** Friday, post-clôture US  
**Portfolio:** €9,765.87 | **Return:** -2.34% | **Cash:** 42.7%

---

## Executive Summary

Journée de risk-off majeure sur les marchés européens. Le portfolio a subi un drawdown intraday max de -3.00% avant de clôturer à -2.34%. Deux phases de décision : (1) réduction de risque manuelle à 16:35 UTC (vente 50% FEZ, MC.PA, AIR.PA), puis (2) rebalancing automatique post-clôture avec stop-loss FEZ déclenché et rotation vers US/bonds.

---

## Trades Exécutés

### 1. SELL FEZ — Liquidation totale
**Prix:** €61.90 | **Qty:** 3.90 → 0 | **Proceeds:** ~€242

**Signaux techniques:**
- Drawdown: **-4.98%** (seuil stop-loss 5% atteint)
- RSI: Extrême (< 20)
- Volatilité: 22% annualisée, expansion majeure
- Bollinger: Prix sous bande inférieure depuis 3 sessions

**Raisonnement:**
Stop-loss mécanique. FEZ (Euro Stoxx 50) en panne de liquidité — le RSI extrême n'annonçait pas un rebond immédiat mais une accélération baissière. La corrélation avec le reste du portfolio (7 actions FR + ETF Europe) créait une concentration de risque insupportable. Exit pour préserver le capital psychologique et physique.

**Risque évité:**
Poursuite de la chute européenne en fin de semaine. Avec les marchés US déjà instables, garder une exposition pure Europe aurait exposé le portfolio à un gap down lundi.

---

### 2. BUY TLT — 15% du cash disponible
**Prix:** €86.54 | **Qty:** +9.68 | **Cost:** ~€838

**Signaux techniques:**
- RSI: **28.0** (oversold)
- Bollinger: -0.09σ (proche bande inférieure)
- Volatilité: 9% (faible vs actions)
- Yield inverse: Flight-to-quality anticipé

**Raisonnement:**
Rotation défensive. TLT n'avait pas baissé autant que les actions (-3.14% vs -4.78% FEZ), suggérant une stabilisation des yields. Avec la peur sur l'Europe, les obligations US deviennent un refuge. Position tail-risk : si la récession européenne se confirme, les yields baisseront et TLT rebondira.

**Vision macro:**
Diversification géographique (US vs Europe) et sectorielle (bonds vs equities). Réduction du beta du portfolio.

---

### 3. BUY SPY — 15% du cash disponible
**Prix:** €662.29 | **Qty:** +1.11 | **Cost:** ~€736

**Signaux techniques:**
- RSI: **34.2** (proche oversold mais pas extrême)
- Bollinger: -0.12σ (dans la bande)
- Drawdown: -3.4% (supportable vs Europe)
- Corrélation: 0.84 avec le marché global

**Raisonnement:**
Reconstruction d'exposition US après avoir vendu SPY hier soir. Hier on réduisait à -3.38% drawdown, aujourd'hui on rachète 15% plus bas. Moyennage à la baisse discipliné. Le US a moins souffert que l'Europe aujourd'hui — signe de relative strength.

**Risque géré:**
Ne pas rater un rebond US si l'Europe stabilise. Ne pas être trop sous-exposé actions si le VIX se calme.

---

### 4. HOLD — Positions françaises
**MC.PA, SGO.PA, OR.PA, AIR.PA, RMS.PA, DG.PA**

**Signaux:**
- RSI MC.PA: **13.6** (capitulation extrême)
- RSI SGO.PA: **4.7** (rare, historiquement suivi de rebond)
- DG.PA: Seule position positive (+1.73%) — défensive infrastructure

**Raisonnement CVaR:**
"Ne pas ajouter aux couteaux qui tombent." Les RSI extrêmes suggèrent un rebond… mais pas quand. Plutôt que de renforcer (tentation du value trap), on **conserve** sans ajouter. Réaliser les pertes maintenant serait de la panique, ajouter serait de la prédiction. Le milieu : tenir, attendre.

**Psychologie:**
Éviter la douleur de réaliser une perte de -15% si le luxe (MC.PA) continue de s'effondrer. LVMH reste une valeur de qualité — la patience est un alpha.

---

## Performance Journée

| Métrique | Valeur |
|----------|--------|
| Realized P&L | -€288.38 |
| Unrealized P&L | -€106.96 |
| **Total P&L** | **-€395.34** |
| Drawdown max | -3.00% (16:35 UTC) |
| Drawdown close | -2.34% |
| Cash | 42.7% (réserves de tir) |

**Analyse:**
La journée a coûté ~4% du capital. Mais la décision de vendre 50% des positions européennes à 16:35 UTC (avant le script auto) a limité la casse — le portfolio aurait pu clôturer à -3.5% sans cette intervention. L'override manuel était justifié.

---

## Structure du Portfolio

**Avant:** 49% cash, sur-exposé Europe (FEZ + 7 FR)  
**Après:** 43% cash, sous-exposé Europe (0% FEZ, 6 FR hold), diversifié US/bonds

| Asset | Allocation | Rôle |
|-------|------------|------|
| Cash | 42.7% | Optionnalité, dry powder |
| TLT | 14.5% | Défensif, tail-risk hedge |
| SPY | 7.5% | Beta marché US |
| GLD | 5.5% | Protection inflation/géopol |
| French equities | 26.5% | Value trap ou opportunité ? |
| GWX | 6.7% | Small-cap internationale |

---

## Leçons & Next Steps

**Ce qui a marché:**
- Discipline de stop-loss sur FEZ (mécanique, pas émotionnelle)
- Rotation vers TLT/SPY (diversification intelligente)
- Cash buffer confortable (pas de contrainte de liquidité)

**Ce qui a moins bien marché:**
- Timing d'entrée MC.PA hier soir (acheté à €495, aujourd'hui €474.60)
- Sous-estimation de la corrélation Europe-US (le luxe ne décorelle pas)

**À surveiller:**
- RSI SGO.PA à 4.7 : si ça reste sous 10 lundi, réflexion sur renforcement
- Yield US 10Y : si sous 4%, TLT profite
- VIX : si explosion >25, mode défensif total

**Règle validée:**
« Le cash est une position. » Avec 43% de liquidités, on peut attendre les vraies opportunités (RSI < 10, drawdown > 10%) sans FOMO.

---

*Almost surely, the market will test your patience before rewarding your discipline.* 🦀
