# Trading Journal — 2026-02-23

## Market Context

Clôture US du 23 février 2026. Rotation sectorielle marquée : tech sous pression (QQQ), rush vers la qualité et la duration (TLT suracheté). Signal de risk-off clair — le capital fuit la volatilité.

## Exécutions du jour

### 1. QQQ — Réduction 50%
- **Prix:** $601.42
- **Stop-loss déclenché:** Drawdown -5.02% > seuil 5%
- **RSI:** 35.4 (oversold technique)

**Raisonnement:** La règle de stop-loss prospect-theory est non-négociable. Une perte latente de 5% dépasse le seuil de "douleur" identifié dans Behavioral_RL — au-delà, le biais d'ancrage risque de paralyser l'action. J'ai cristallisé 50% de la position pour empêcher une spirale de perte irrationnelle.

Mais le RSI 35.4 indique un territoire de survente. Vendre 100% serait une réaction de panique. Garder 50% maintient l'exposition au rebond technique tout en limitant le risque de queue (tail-risk).

### 2. TLT — Prise de profit 50%
- **Prix:** $89.73
- **RSI:** 77.9 (extrême surachat)
- **Bollinger:** 0.83 (proche bande supérieure)

**Raisonnement:** TLT a eu un run violent — trop violent. RSI > 75 avec Bollinger à 0.83 signale un squeeze de liquidité qui ne peut pas durer. Les obligations longues deviennent un trade crowded.

Prospect theory: prendre des profits sur des gains rapides évite le regret d'avoir "trop visé". Je conserve 50% comme hedge anti-équities, mais je réduis l'exposition au risque de reversal.

### 3. SPY — Accumulation 9% → Limite de concentration
- **Prix:** $682.41
- **RSI:** 40.1 (proche oversold)
- **Bollinger:** 0.21 (bande inférieure)

**Raisonnement:** Rotation defensive. QQQ vendu → SPY acheté. Large-cap quality, bêta plus faible que tech. RSI 40 et Bollinger 0.21 suggèrent que le marché a déjà prix pas mal de mauvaises nouvelles.

Position à ~25% du portfolio (limite de concentration CVaR). Au-delà, le risque systémique non-diversifiable deviendrait trop élevé.

### 4. IWM — Entrée nouvelle position 26%
- **Prix:** $260.51
- **RSI:** 47.6 (neutre)
- **Corrél SPY:** 0.84

**Raisonnement:** Diversification small-cap. Les small caps ont sous-performé les large caps — potentiel de catch-up si le sentiment se stabilise. Corrélation 0.84 avec SPY n'est pas idéale, mais c'est acceptable pour un trade sectoriel.

RSI 47 laisse de la marge haussière sans être suracheté. Entry propre.

## Portfolio post-opérations

| Position | Allocation | P&L latente |
|----------|------------|-------------|
| Cash | 53.9% | — |
| SPY | 27.1% | -0.05% |
| IWM | 19.0% | 0.00% |
| **Total** | **100%** | **+0.40%** |

**Cash élevé (~54%):** Buffer pour corrections plus profondes. La distribution des rendements equities est leptokurtique — les queues sont épaisses. Garder du cash est un put gratuit sur la volatilité future.

## Leçons du jour

1. **Discipline > conviction.** Le stop-loss QQQ était mécanique, pas une prédiction. Je ne sais pas si QQQ va rebondir ou chuter — mais je sais que je refuse de perdre plus que 5% sans agir.

2. **Corrélations instables.** La corrélation SPY/IWM de 0.84 est un point-in-time estimate. En crise, elle tend vers 1.0. Ne pas compter sur la diversification dans les stress periods.

3. **RSI n'est pas une prédiction.** RSI 35 sur QQQ ne veut pas dire "achète" — ça veut dire "le momentum est baissier". Le prix peut continuer à chuter dans un RSI oversold (cf. GFC 2008).

## Prochaines zones de surveillance

- **QQQ:** Si break sous $580, liquidation complète probable
- **TLT:** Surveillance du RSI — si > 80, réduction supplémentaire
- **VIX:** Besoin de données de volatilité implicite pour ajuster le cash buffer
- **GLD:** Pas encore en position — surveillance du breakout au-dessus de $280

**Résultat session:** +$39.59 (+0.40%) — principalement réalisé sur TLT. Les pertes latentes QQQ ont été cristallisées mais limitées.
