# paris-road-safety

Analyse exploratoire croisant les **accidents de la circulation géolocalisés** et le **trafic routier** à Paris, pour tester une hypothèse simple : *le danger d'un axe est-il proportionnel à son trafic ?*

## Objectif
Tester, sur le réseau routier parisien instrumenté, si **le risque d'accident par véhicule** dépend du niveau de trafic de l'axe.

## Données
Trois jeux open-data de la Ville de Paris, publiés sur [opendata.paris.fr](https://opendata.paris.fr/) :

| Jeu | Portail | Licence |
|---|---|---|
| **Accidents de la circulation** (`accidentologie0`, base BAAC géolocalisée) | https://opendata.paris.fr/explore/dataset/accidentologie/ | Open Database License (ODbL) |
| **Comptages routiers permanents** (volumes de trafic agrégés par capteur) | https://opendata.paris.fr/explore/dataset/comptages-routiers-permanents/ | Open Database License (ODbL) |
| **Référentiel des comptages routiers** (métadonnées + coordonnées des capteurs) | https://opendata.paris.fr/explore/dataset/referentiel-comptages-routiers/ | Open Database License (ODbL) |

Pas de données personnelles ni confidentielles — les accidents sont agrégés par
localisation, les capteurs mesurent uniquement des volumes anonymes.

## Méthode

1. **Cadrage** — hypothèse de travail, question métier, périmètre géographique.
2. **Audit** — exploration des trois jeux, qualité des champs clés (`annee`, coordonnées, gravité).
3. **Faisabilité** — vérification que les trois jeux se joignent géographiquement.
4. **Nettoyage** :
   - Doublons stricts et quasi-doublons
   - Valeurs manquantes et cohérence de la gravité (32 lignes incohérentes repérées)
   - Coordonnées hors périmètre parisien
5. **Jointure spatiale** — rapprochement accidents ↔ capteur le plus proche via `scipy.spatial.cKDTree`, seuil 150 m. **24 840 accidents rattachés (74 %)**, 8 796 écartés (26 %). Biais déclaré : l'étude porte sur le **réseau structurant** (grands axes), pas sur les rues résidentielles non instrumentées.
6. **Fiabilisation du dénominateur** — retrait de 49 capteurs à débit < 50 véh/h (mesures implausibles).
7. **Variables analytiques** — `risque = accidents_distincts / (q_moy / 1000)` (accidents pour 1 000 véh/h), découpage en 4 tiers de trafic.
8. **Tests statistiques** — Kruskal-Wallis entre tiers + corrélation de Spearman trafic / risque.
9. **Dashboard Power BI** — restitution (`dashboard/securite_routiere_paris.pbix`).

## Stack
- Python, pandas, numpy, requests
- scipy (`spatial.cKDTree` pour la jointure spatiale)
- matplotlib, Folium (carte interactive)
- **Power BI** (dashboard final)

## Résultats

### Conclusion de l'analyse
Le risque d'accident **par véhicule** décroît fortement quand le trafic augmente :

| Tier de trafic | Débit moyen (véh/h) | Accidents (absolu) | Risque / 1 000 véh |
|---|---|---|---|
| T1 (faible) | 139 | 8,4 | 48,2 |
| T2 | 244 | 9,9 | 30,4 |
| T3 | 398 | 11,3 | 22,9 |
| T4 (fort) | 1 555 | 15,2 | 9,8 |

- **En absolu**, les accidents augmentent avec le trafic (8,4 → 15,2 par capteur) : attendu.
- **Par véhicule**, le risque s'effondre (48 → 10 pour 1 000 véh, ~5× moins), de manière monotone.
- Confirmé statistiquement : **Kruskal-Wallis H ≈ 360, p ≈ 8e-78** et **Spearman ρ = −0,46, p ≈ 3e-100**.
- **Hypothèse de départ réfutée** : le danger d'un axe n'est **pas** proportionnel à son trafic. Les axes à fort débit (périphérique) sont surtout **séparés des piétons, sans intersections, à vitesse régulée** — c'est cette **configuration** qui réduit le risque par véhicule, pas le volume lui-même. Le trafic est un **marqueur** de la hiérarchie des voies, pas un facteur causal.

### Dashboard Power BI
Fichier `dashboard/securite_routiere_paris.pbix`, 1 page (1280×720), 6 visuels :

1. **Clustered column chart** — « Le risque par véhicule chute quand le trafic augmente »
   (axe X : `tier` ; valeurs : `risque_1000veh`)
2. **Carte** — « Intensité de trafic par axe » (lat/lon des capteurs, série = tier de trafic)
3. **Carte** — « Risque par véhicule (vert = sûr) » (lat/lon des capteurs, série = catégorie de risque)
4. **Slicer** sur la gravité des accidents (`accidents_geocodes.gravite`)
5. **2 zones de texte** (titres / commentaires)

Tables utilisées dans le modèle : `capteurs_risque` (tier, risque_1000veh, lat, lon, Catégorie risque) et `accidents_geocodes` (gravite).

## Comment lancer

```bash
python -m venv venv
source venv/bin/activate      # Windows : venv\Scripts\activate
pip install pandas numpy scipy matplotlib folium requests jupyter

jupyter notebook accidentologie_trafic_paris.ipynb
```

Les trois jeux open-data se téléchargent via les APIs OpenData Paris (URLs dans le notebook).

Le dashboard Power BI est fourni dans `dashboard/securite_routiere_paris.pbix` — ouvrir avec **Power BI Desktop**.
