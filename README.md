# MetaSafe Industrial

Tableau de bord de maintenance prédictive pour un parc de machines industrielles.
MetaSafe Industrial est une application web développée avec Streamlit. Elle analyse les données capteurs d'un parc de machines et estime, grâce à un modèle d'apprentissage automatique, la probabilité qu'une machine tombe en panne. L'objectif est d'anticiper les défaillances pour planifier la maintenance avant l'arrêt de production.

## 🎥 Démo vidéo

▶️ **[Regarder la démo sur Google Drive](https://drive.google.com/drive/folders/1fHVSTtVsDsgb3VACNhxlLEXfmiarlOnJ?usp=sharing)**

## Sommaire

- [Fonctionnalités](#fonctionnalités)
- [Jeu de données](#jeu-de-données)
- [Modèle de prédiction](#modèle-de-prédiction)
- [Structure du projet](#structure-du-projet)
- [Installation et lancement](#installation-et-lancement)
- [Authentification Keycloak (optionnelle)](#authentification-keycloak-optionnelle)
- [Ré-entraîner le modèle](#ré-entraîner-le-modèle)
- [Dépannage](#dépannage)
- [Technologies utilisées](#technologies-utilisées)
- [Équipe du projet](#équipe-du-projet)

## Fonctionnalités

L'application est organisée en cinq pages, accessibles depuis le menu latéral.

| Page | Contenu |
|---|---|
| **Vue d'ensemble** | Indicateurs clés (parc surveillé, nombre de pannes, disponibilité, usure et couple moyens), répartition sain / panne, pannes par qualité de machine, décomposition des modes de défaillance. |
| **Distributions** | Pour chaque variable capteur : histogramme, densité lissée (KDE), boîte à moustaches et distribution par qualité de machine. Calcul du pouvoir discriminant de la variable (d de Cohen). |
| **Analyse multivariée** | Matrice de corrélation, carte des zones de contrainte mécanique (vitesse × couple), écart de température process – air, puissance mécanique. |
| **Modes de défaillance** | Synthèse des 5 modes de panne, analyse détaillée d'un mode (profil moyen des machines concernées, écarts relatifs) et recommandations de maintenance. |
| **Diagnostic prédictif** | Saisie des mesures d'une machine, probabilité de défaillance calculée par le modèle, verdict (conforme / risque modéré / risque élevé) et analyse des facteurs de risque. |

Autres fonctionnalités :

- Filtre global par qualité de machine (L, M, H) appliqué à toutes les pages d'analyse.
- Encadrés d'interprétation générés automatiquement à partir des données.
- Authentification SSO via Keycloak, avec gestion des rôles (Administrateur, Analyste, Opérateur). Sans configuration, l'application démarre en mode local.
- Interface sombre et sobre, adaptée à un usage en salle de supervision.



## Jeu de données

Le projet utilise le jeu de données public **AI4I 2020 Predictive Maintenance Dataset** (UCI Machine Learning Repository), fourni dans le fichier `ai4i2020.csv`.

- 10 000 observations, une par machine.
- 339 pannes (3,39 %) : les classes sont fortement déséquilibrées.

| Variable | Description |
|---|---|
| `Type` | Qualité de la machine : L (Low, 60 %), M (Medium, 30 %), H (High, 10 %) |
| `Air temperature [K]` | Température de l'air ambiant |
| `Process temperature [K]` | Température du procédé |
| `Rotational speed [rpm]` | Vitesse de rotation (tr/min) |
| `Torque [Nm]` | Couple (N·m) |
| `Tool wear [min]` | Usure de l'outil (minutes) |
| `Machine failure` | Variable cible : 1 si la machine est en panne |

Une panne peut avoir une ou plusieurs causes, appelées modes de défaillance :

| Code | Mode | Condition de déclenchement |
|---|---|---|
| TWF | Usure de l'outil | Usure comprise entre 200 et 240 min |
| HDF | Dissipation de chaleur | Écart process – air < 8,6 K et vitesse < 1 380 tr/min |
| PWF | Défaut de puissance | Puissance mécanique hors de la plage 3 500 – 9 000 W |
| OSF | Surcontrainte | Couple × usure > 11 000 (L), 12 000 (M) ou 13 000 (H) |
| RNF | Panne aléatoire | Environ 0,1 % de probabilité, sans cause identifiable |

## Modèle de prédiction

Le modèle est entraîné dans le notebook `entrainement.ipynb`.

- **Préparation** : encodage de la qualité de machine (`LabelEncoder`). Les identifiants (`UDI`, `Product ID`) et les colonnes de modes de panne sont retirés, pour que le modèle n'apprenne pas directement la réponse.
- **Variables d'entrée** : qualité, température air, température process, vitesse de rotation, couple, usure outil.
- **Découpage** : 80 % entraînement (8 000 lignes) / 20 % test (2 000 lignes), stratifié sur la cible.
- **Rééquilibrage** : SMOTE, appliqué uniquement sur les données d'entraînement. Il crée des pannes synthétiques pour atteindre un équilibre 50 / 50.
- **Algorithme** : Random Forest (100 arbres).

Résultats sur l'ensemble de test, tels qu'ils figurent dans le notebook :

| Classe | Précision | Rappel | F1-score |
|---|---|---|---|
| Sain | 0,99 | 0,97 | 0,98 |
| Panne | 0,43 | 0,72 | 0,54 |
| **Exactitude globale** | | | **0,96** |

Le modèle détecte environ 72 % des pannes réelles. En contrepartie, il produit un certain nombre de fausses alertes, ce qui reste acceptable en maintenance préventive : une inspection inutile coûte moins cher qu'un arrêt imprévu.

Le modèle et l'encodeur sont enregistrés dans `modele_maintenance_predictive.pkl` et `label_encoder_type.pkl`, puis chargés par l'application.

## Structure du projet

```
MetaSafe-Industrial/
├── app_auth.py                        Application Streamlit (5 pages)
├── entrainement.ipynb                 Notebook d'analyse et d'entraînement du modèle
├── ai4i2020.csv                       Jeu de données AI4I 2020
├── modele_maintenance_predictive.pkl  Modèle Random Forest entraîné
├── label_encoder_type.pkl             Encodeur de la qualité de machine (L / M / H)
├── requirements.txt                   Dépendances Python
├── secrets_template.toml              Modèle de configuration Keycloak
├── .streamlit/
│   └── config.toml                    Thème de l'interface
├── docs/
│   └── screenshots/                   Captures d'écran utilisées dans ce README
├── .gitignore
└── README.md
```

## Installation et lancement

### Prérequis

Python 3.10 à 3.13 (Python 3.12 est recommandé). Sous Windows, cochez « Add Python to PATH » pendant l'installation.

### Première installation

Ouvrez un terminal dans le dossier du projet, puis exécutez les commandes suivantes.

**Windows**

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
streamlit run app_auth.py
```

**macOS / Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
streamlit run app_auth.py
```

L'application s'ouvre automatiquement dans le navigateur à l'adresse http://localhost:8501.

### Lancements suivants

```bash
.venv\Scripts\activate          # macOS / Linux : source .venv/bin/activate
streamlit run app_auth.py
```

Pour arrêter l'application, appuyez sur `Ctrl + C` dans le terminal.

## Authentification Keycloak (optionnelle)

Sans configuration, l'application démarre en mode local : l'accès est direct, sans connexion. Pour activer l'authentification SSO :

1. Copiez `secrets_template.toml` vers `.streamlit/secrets.toml`.
2. Renseignez les valeurs de votre serveur :

   ```toml
   [keycloak]
   url       = "https://auth.mon-entreprise.com"
   realm     = "industrial-realm"
   client_id = "industrial-ai-insights"
   ```

3. Dans la console d'administration Keycloak, configurez le client :
   - type OpenID Connect, authentification client désactivée (client public) ;
   - Valid redirect URIs : `http://localhost:8501/*` ;
   - Web origins : `http://localhost:8501`.
4. Créez les rôles de realm et attribuez-les aux utilisateurs :

   | Rôle Keycloak | Rôle dans l'application |
   |---|---|
   | `admin` | Administrateur |
   | `analyste` | Analyste |
   | aucun des deux | Opérateur |

Sur Streamlit Community Cloud, collez le même contenu dans **Settings → Secrets** de l'application.

> ⚠️ Le fichier `.streamlit/secrets.toml` contient des informations sensibles : il est exclu du dépôt par le `.gitignore` et ne doit jamais être publié.

## Ré-entraîner le modèle

1. Installez les dépendances supplémentaires :

   ```bash
   pip install jupyter seaborn matplotlib imbalanced-learn
   ```

2. Ouvrez le notebook :

   ```bash
   jupyter notebook entrainement.ipynb
   ```

3. Exécutez toutes les cellules. Les fichiers `modele_maintenance_predictive.pkl` et `label_encoder_type.pkl` sont régénérés dans le dossier du projet.

Si vous changez la version de scikit-learn, mettez aussi à jour la ligne correspondante dans `requirements.txt`, pour que l'application charge le modèle avec la même version que celle de l'entraînement.

## Dépannage

| Problème | Solution |
|---|---|
| `python` n'est pas reconnu | Réinstallez Python en cochant « Add Python to PATH », ou utilisez `py` à la place de `python`. |
| `streamlit` n'est pas reconnu | L'environnement virtuel n'est pas activé : exécutez `.venv\Scripts\activate`. |
| Erreur lors de l'installation de scikit-learn | Votre version de Python est trop récente. Installez Python 3.12. |
| Le port 8501 est déjà utilisé | Lancez l'application sur un autre port : `streamlit run app_auth.py --server.port 8502`. |
| Erreur de chargement des ressources | Vérifiez que les fichiers `.pkl` et `ai4i2020.csv` sont bien dans le même dossier que `app_auth.py`. |

## Technologies utilisées

- **Streamlit** : interface web
- **Plotly** : graphiques interactifs
- **pandas, NumPy, SciPy** : traitement des données et statistiques
- **scikit-learn** : modèle Random Forest
- **imbalanced-learn** : rééquilibrage SMOTE (entraînement uniquement)
- **streamlit-keycloak** : authentification SSO


