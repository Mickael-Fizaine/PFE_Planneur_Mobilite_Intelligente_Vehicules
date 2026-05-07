# Planneur de Mobilité Intelligente pour Véhicules Électriques

> **Projet de Fin d'Études (PFE)** — Optimisation multi-objectif de trajets pour une flotte de véhicules électriques via la métaheuristique ALNS (Adaptive Large Neighborhood Search) et un protocole de négociation multi-agents, sur un réseau routier réel.

---

## Présentation

Ce projet traite du **problème de planification de trajets pour véhicules électriques (EVRPP)** — optimiser les itinéraires d'une flotte de véhicules électriques tout en gérant les contraintes de batterie, la tarification dynamique des bornes de recharge et la disponibilité des connecteurs. Trois approches ont été implémentées et comparées :

| Approche | Description |
|---|---|
| **Algorithme Naïf** | Référence greedy — routage direct avec sélection de la première borne disponible |
| **ALNS Optimisation** | Recherche à grands voisinages adaptatifs avec coordination globale de la flotte |
| **ALNS + Négociation** | ALNS étendu avec un protocole de négociation de prix entre véhicules et bornes |

Les algorithmes sont testés sur un réseau routier réel extrait d'OpenStreetMap, en simulant jusqu'à **50 véhicules électriques** avec des capacités de batterie, des taux de consommation, des heures de départ et des profils d'optimisation hétérogènes.

---

## Fonctionnalités principales

- **Routage géospatial réel** — réseau routier chargé depuis OpenStreetMap via `osmnx` ; plus courts chemins calculés avec NetworkX sur le graphe routier réel
- **Fonction de coût multi-objectif** — combinaison pondérée du temps de trajet, de la consommation énergétique et du coût de recharge, paramétrée par 4 profils conducteur :
  - `urgent` — priorité au temps de trajet (60%)
  - `econome` — priorité au coût de recharge (60%)
  - `vert` — priorité à l'efficacité énergétique (60%)
  - `equilibrer` — pondération égale entre tous les objectifs
- **Tarification dynamique** — les prix des bornes suivent une variation sinusoïdale sur 24 heures, encourageant la recharge aux heures creuses
- **Planification de l'occupation des connecteurs** — suivi par créneaux horaires pour éviter les conflits de connecteurs au sein de la flotte
- **Optimisation globale de la flotte** — les véhicules sont réoptimisés de manière itérative en partageant un planning de recharge global jusqu'à convergence
- **Protocole de négociation de prix** — chaque véhicule compare sa borne optimale à l'alternative la plus rapide et propose un alignement de prix si la remise est inférieure à 20%
- **Export de cartes interactives** — trajets de la flotte visualisés en HTML (Folium) avec chemins colorés par véhicule, marqueurs de bornes et informations en popup

---

## Architecture du projet

```
PFE_Planneur_Mobilite_Intelligente_Vehicules/
├── Rendu_PFE/
│   ├── Code/
│   │   ├── Algo_Naif/
│   │   │   └── Algo_Naif.ipynb                          # Algorithme de référence (greedy)
│   │   ├── Algo_Optimisation/
│   │   │   ├── PFE_Mobilite_Vehicule_Optimisation.ipynb # ALNS — optimisation globale de flotte
│   │   │   └── Graphiques/                              # Graphiques générés (zippés)
│   │   └── Algo_Negociation/
│   │       ├── PFE_Mobilite_Vehicule_Negociation.ipynb  # ALNS + protocole de négociation
│   │       └── Graphiques/                              # Graphiques générés (zippés)
│   ├── Powerpoint/
│   │   └── PFE_Presentation.pdf                         # Diaporama de présentation
│   └── Rapport/
│       └── PFE_03_Rapport.pdf                           # Rapport technique final
```

---

## Stack technique

| Catégorie | Outils |
|---|---|
| Langage | Python 3.x |
| Géospatial | `osmnx 2.0.2`, `geopandas 1.0.1`, `shapely 2.0.7`, `pyproj` |
| Algorithmes de graphes | `networkx 3.4.2` |
| Visualisation | `folium 0.19.5`, `matplotlib`, `branca` |
| Traitement de données | `pandas 2.2.2`, `numpy` |
| Environnement | Jupyter Notebook |

---

## Installation

### Prérequis

- Python 3.9+
- `pip` ou `conda`

### Mise en place

```bash
# Cloner le dépôt
git clone https://github.com/cybermika02/PFE_Planneur_Mobilite_Intelligente_Vehicules.git
cd PFE_Planneur_Mobilite_Intelligente_Vehicules

# Installer les dépendances
pip install osmnx==2.0.2 networkx==3.4.2 folium==0.19.5 geopandas==1.0.1 \
            shapely==2.0.7 matplotlib pandas==2.2.2 numpy requests
```

> **Note :** `osmnx` nécessite `libspatialindex` sur certains systèmes. Sous Ubuntu/Debian : `sudo apt-get install libspatialindex-dev`. Sous Windows, l'installation via `conda` est recommandée pour simplifier la gestion des dépendances.

### Lancer les notebooks

```bash
# Démarrer Jupyter
jupyter notebook

# Puis ouvrir l'un des fichiers suivants :
# Rendu_PFE/Code/Algo_Naif/Algo_Naif.ipynb
# Rendu_PFE/Code/Algo_Optimisation/PFE_Mobilite_Vehicule_Optimisation.ipynb
# Rendu_PFE/Code/Algo_Negociation/PFE_Mobilite_Vehicule_Negociation.ipynb
```

Exécuter toutes les cellules de haut en bas. Les notebooks sont autonomes — ils téléchargent le réseau routier OSM à l'exécution, génèrent la flotte de véhicules, lancent l'optimisation et exportent les résultats.

---

## Algorithmes

### ALNS (Adaptive Large Neighborhood Search)

L'optimiseur itère une boucle **destruction → réparation** :

```
Solution initiale
    ↓
Destruction : suppression aléatoire de 30 à 40 % des nœuds intermédiaires
    ↓
Réparation : réinsertion des nœuds supprimés aux positions de moindre coût
             (recherche exacte par permutation pour ≤ 4 nœuds, greedy sinon)
    ↓
Acceptation si amélioration  →  itération jusqu'à convergence
```

La faisabilité du trajet est vérifiée à chaque étape : une solution est valide uniquement si le véhicule peut atteindre chaque nœud successif sans dépasser sa capacité de recharge.

### Optimisation globale de la flotte

```
Pour chaque cycle d'optimisation :
    Pour chaque véhicule v de la flotte :
        Lancer ALNS(v) avec le planning de recharge global courant
        Mettre à jour le planning avec les créneaux de recharge de v
    Si le coût total de la flotte s'est amélioré → continuer
    Sinon → convergence
```

Ce schéma itératif résout les conflits entre véhicules qui se disputent les mêmes connecteurs de recharge au même moment.

### Protocole de négociation

```
Le véhicule sélectionne la borne optimale S* depuis son trajet ALNS
    ↓
Identification de la borne alternative la plus rapide S_alt
    ↓
Si S* est occupée → sélectionner S_alt directement
Si remise(S*, S_alt) ≤ 20% → proposer un alignement de prix à S*
Si remise > 20% → rejeter la négociation, conserver S*
```

La négociation réduit la disparité de prix entre les bornes en concurrence et modélise une dynamique de marché réaliste.

---

## Résultats et sorties

Chaque notebook génère :

- **`carte_routes_vehicules.html`** — carte Folium interactive avec trajets colorés par véhicule, marqueurs de bornes (rouge = utilisée, bleu = non utilisée) et détails en popup
- **Graphiques par véhicule** — prix payé, énergie consommée et temps de trajet à chaque arrêt de recharge
- **Graphiques de synthèse de la flotte** — distribution des heures de départ, coût objectif par véhicule, niveaux de batterie finaux
- **Courbes de tarification dynamique** — variation des prix sur 24 heures par borne

### Scénario de test

| Paramètre | Valeur |
|---|---|
| Itinéraire | Tour Eiffel, Paris → Houilles (~13 km) |
| Bornes de recharge | 20 (réparties le long du corridor) |
| Taille de la flotte | 50 véhicules (Optimisation) / 10 véhicules (Négociation) |
| Capacité batterie | 40 – 100 kWh |
| Consommation | 0,15 – 0,30 kWh/km |
| Horizon temporel | 24 heures avec tarification dynamique |

---

## Documentation

- [Rapport technique final](Rendu_PFE/Rapport/PFE_03_Rapport.pdf) — méthodologie complète, formulation mathématique et analyse des résultats
- [Diaporama de présentation](Rendu_PFE/Powerpoint/PFE_Presentation.pdf) — synthèse du projet et résultats visuels

---

## Contexte académique

Ce projet a été réalisé dans le cadre d'un **Projet de Fin d'Études (PFE)** à CY TECH. Il illustre une application concrète de :

- **Optimisation combinatoire** — métaheuristique ALNS
- **Théorie des graphes** — plus court chemin, routage sur réseau
- **Calcul géospatial** — données OpenStreetMap, réseaux routiers réels
- **Théorie des jeux** — protocole de négociation bilatérale
- **Systèmes multi-agents** — coordination de flotte avec planification de ressources partagées

---

## Auteur

**Mickael Fizaine**
- GitHub : [@cybermika02](https://github.com/cybermika02)

---

## Licence

Ce projet est à vocation académique et portfolio. Vous êtes libre d'explorer le code et les méthodologies.
