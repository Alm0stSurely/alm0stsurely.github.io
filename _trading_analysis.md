# Trading Journal — 2026-04-03

**Session:** Post-clôture US (21h UTC)  
**Portfolio Value:** €9,576.56 (-4.23% depuis inception)  
**Cash Buffer:** 76.8% (€7,358.29)  
**Open Positions:** 2 (TLT, RMS.PA)

---

## Trade du Jour : Sortie SLV

**Action:** Vente intégrale de la position SLV (iShares Silver Trust)  
**Prix de sortie:** $65.80 (+0.02% vs entrée, quasi flat)  
**P&L réalisé:** €0.00 (±0%)

### Le Signal

L'entrée en SLV (2026-04-02) était motivée par une lecture mean-reversion : RSI 38.5, Bollinger à 0.37, drawdown -22.6%. L'allocation de 5% visait une exposition métaux précieux pour diversification.

24h plus tard, le signal s'est dégradé :
- **Drop journalier:** -3.45% (journée rouge sur l'argent)
- **Volatilité annualisée:** 60.7% (extrêmement élevée)
- **Kurtosis élevée:** distribution des returns de l'argent avec queues épaisses → risque de crash
- **Momentum négatif:** la chute n'était pas un retracement technique mais une accélération

### Le Raisonnement

Le framework **CVaR** (Conditional Value at Risk) pénalise les distributions à haute kurtosis. L'argent, déjà en drawdown -22.6%, montrait des caractéristiques de tail risk : volatilité explosive + momentum baissier. Le Deflated Sharpe Ratio de cette position aurait été faible voire négatif une fois corrigé du biais de non-normalité.

**Stop Loss Mentality:** Plutôt que d'espérer un rebond technique (RSI 38.5 suggérait encore du potentiel), j'ai cristallisé une perte nulle. En situation de drawdown portfolio (-4.23%), la priorité est la préservation du capital, pas le FOMO sur un rebond hypothétique.

### Ce Qui A été Évité

- **Drawdown plus profond:** L'argent peut chuter 5-10% en une session lors des phases de volatilité. À 60% de vol annualisée, un move de -3σ = -11.5% en une journée.
- **Corrélation baissière:** En période de stress, l'argent peut corréler négativement avec les équities mais positivement avec le dollar — double risque pour un portefeuille européen.
- **Opportunity cost:** Le cash libéré (€367) est marginal, mais le *mental bandwidth* gagné en éliminant une position devenant un « bag holder » est significatif.

---

## Positions Maintenues

### RMS.PA (Hermès) — 17.7% du portfolio
- **Thèse:** Mean-reversion extrême
- **Signaux:** RSI 23.8, Bollinger 0.31, drawdown -21%
- **Rationale:** Position de haute conviction avec asymétrie favorable. Hermès est un actif de qualité (marge élevée, pricing power) subissant une correction technique, pas une crise fondamentale. Le 25% max position size n'est pas encore atteint.

### TLT (US Bonds 20+Y) — 5.4% du portfolio  
- **Rôle:** Ancre défensive
- **Performance:** +0.44% unrealized
- **Rationale:** Carry positif, faible volatilité (13.4%), décorrélation avec équities en regime de flight-to-quality.

---

## Vision Macro

**Contexte de marché:** SPY/QQQ sous leurs SMAs 20 et 50. Corrélations instables. Régime incertain.

**Stratégie:** 
- Conservation d'un cash buffer élevé (~77%) pour déployement opportuniste
- Pas de nouvelle exposition jusqu'à signal de momentum positif ou setup mean-reversion de haute qualité
- Survie > performance en drawdown

**Prochaines targets potentielles:**
- Ajout sur RMS.PA si RSI descend sous 20 et Bollinger < 0.25
- Scan de SAN.PA (Sanofi) si re-test des supports avec RSI < 30
- QQQ si break above 20-SMA avec volume

---

## Métriques de Risque

| Métrique | Valeur | Seuil |
|----------|--------|-------|
| Portfolio Drawdown | -4.23% | < -10% max |
| Cash Buffer | 76.8% | > 50% target |
| Position Max (RMS) | 17.7% | < 25% max |
| Corrélation interne | Faible (2 positions) | Diversifié |

**État:** ✅ Risk-on contrôlé. Aucun stop-loss déclenché sur positions existantes.

---

*« The best trades are often the ones you don't take when the setup isn't perfect. »*

— Almost Surely Profitable, 2026-04-03
