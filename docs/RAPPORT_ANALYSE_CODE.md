# Rapport d'analyse du code — KadiPy

**Projet :** KadiPy — « le pandas de l'agriculture africaine »
**Version analysée :** 1.0.0
**Date :** 12 août 2026
**Périmètre :** ~11 800 lignes de Python (`kadi/`), 238 tests unitaires,
3 workflows CI, analyse module par module.

---

## 1. Synthèse

KadiPy est une bibliothèque Python **mature, bien architecturée et bien
testée** pour l'analyse de données agricoles au Bénin. L'architecture en
trois modules (façades `WeatherSession`, `Market`, `DataPipeline`), la
documentation abondante et la stratégie « offline-first » sont d'excellente
facture. **Les 238 tests unitaires passent.**

Cette seconde passe, plus approfondie, a examiné la logique métier ligne à
ligne. Elle **ne révèle aucun bug bloquant**, mais met au jour un ensemble
cohérent de **défauts de configuration et de modélisation** à corriger : une
double source de vérité pour les taux de change, plusieurs clés de
configuration mortes, un facteur de consommation carburant ignoré, et une
hypothèse implicite de données journalières dans le modèle de prévision.

| Critère | Évaluation |
|---|---|
| Architecture | ★★★★★ Séparation des responsabilités exemplaire |
| Lisibilité / documentation | ★★★★★ Docstrings complètes, code commenté |
| Tests | ★★★★☆ 238 tests OK ; intégration réseau non couverte hors ligne |
| Gestion d'erreurs | ★★★★☆ Hiérarchie dédiée, fallbacks robustes |
| Cohérence config / modélisation | ★★★☆☆ Config morte + doublons + caveats modèle |
| Sécurité | ★★★★★ Aucune fuite de secret, OIDC, permissions CI minimales |

---

## 2. Architecture

```
kadi/
├── __init__.py        # Version, logger racine, set_verbosity()
├── config.py          # Configuration centralisée (CONFIG, URLs, taux)
├── exceptions.py      # Hiérarchie KadiException (+ exceptions kidas)
├── cache.py           # Cache SQLite global
├── _utils/            # Réseau (retry), coordonnées GPS
├── _sources/          # Connecteurs bas niveau (Open-Meteo, CHIRPS, SoilGrids)
├── weather/           # Façade WeatherSession : data, location, phenology,
│                      #                          hydrology, risk
├── market/            # Façade Market : pricing, forecasting, logistics,
│                      #   decision_support, data_ingestion, backtesting
└── kidas/             # Façade DataPipeline : cleaner, validator, normalizer,
                       #   cache, sources/{csv,excel,json,netcdf,api}
```

**Points forts confirmés :**

- **Pattern façade** homogène et **injection de dépendances** propre
  (`Market` injecte `pricing`/`logistics`/`weather_session` dans
  `DecisionSupport`). Testable et découplé.
- **Initialisation paresseuse** des composants météo (`_ensure_components`)
  et des données (`_ensure_data`) : économie d'appels réseau.
- **Imports conditionnels** bien gérés (ex. NetCDF/xarray optionnel dans
  `kidas/pipeline.py:25-30`, avec drapeau `_NETCDF_DISPONIBLE`).
- **Mode dégradé transparent** : drapeau `is_simulated` et `confidence_score`
  propagés de bout en bout ; l'utilisateur connaît toujours la fiabilité.
- **Robustesse réseau** : retry + backoff exponentiel, distinction erreurs
  temporaires (429/5xx) vs permanentes (401/403/404), cascades de fallback
  (OSRM → Haversine → valeur par défaut).

---

## 3. Constats détaillés

Classés par sévérité. Chaque constat indique le fichier, la ligne et l'impact.

### 🟠 3.1 — Double source de vérité pour les taux de change (config morte)

- **Où :** `config.py:212-215` vs `market/_normalization.py:175-178`,
  utilisé dans `market/pricing.py:19,126,131`.
- **Constat :** `config.EXCHANGE_RATES` (`XOF_USD = 0.0016`, `XOF_EUR = 0.0015`)
  **n'est jamais lu** par le code. La normalisation des prix utilise une
  **autre** table, `EXCHANGE_RATES_DEFAULT` (`USD_TO_XOF = 620.0`,
  `EUR_TO_XOF = 655.957`), définie dans `_normalization.py`. Le commentaire
  `pricing.py:18` (« remplacés par config.EXCHANGE_RATES si disponibles »)
  décrit un mécanisme qui **n'existe pas**.
- **Impact :** un mainteneur qui met à jour `config.EXCHANGE_RATES` (comme
  l'invite le commentaire « Mise à jour quotidienne prévue ») croira changer
  le taux de conversion sans aucun effet réel. Les deux tables emploient en
  plus des conventions inverses (XOF→USD vs USD→XOF), source de confusion.
- **Recommandation :** supprimer `config.EXCHANGE_RATES`, **ou** brancher
  réellement `pricing._EXCHANGE_RATES` dessus. Une seule source de vérité.

### 🟠 3.2 — Facteur de consommation carburant ignoré

- **Où :** `config.py:135` (`consommation_l_per_100km = 12.0`) vs
  `market/logistics.py:544`.
- **Constat :** la clé de config `consommation_l_per_100km` **n'est utilisée
  nulle part**. Le coût carburant est calculé par
  `d_ab * (gamma_effectif * prix_carburant / 100.0 + mu_checkpoints)`, où le
  `/ 100.0` codé en dur revient à supposer une consommation d'environ
  1,2 L/100 km (via `gamma ≈ 1.2`), très loin des 12 L/100 km déclarés.
- **Impact :** le poste « carburant » du coût de transfert est
  vraisemblablement **sous-estimé d'un ordre de grandeur (~×10)** si
  l'intention était bien 12 L/100 km. Cela fausse `calculate_transfer_cost()`
  et donc les recommandations d'arbitrage spatial.
- **Recommandation :** clarifier la formule et, soit utiliser
  `consommation_l_per_100km` (`coût_km = prix * conso / 100`), soit retirer la
  clé morte et documenter le sens réel de `gamma_route`.

### 🟠 3.3 — Modèle de prévision : hypothèse implicite de données journalières

- **Où :** `market/forecasting.py:78-89` (features) et `:314` (horizon).
- **Constat :** les harmoniques saisonnières utilisent l'**indice
  d'observation** `t = 0,1,2,…` comme s'il s'agissait de **jours**
  (`2π·t/365`, `2π·t/182.5`), et l'horizon futur est
  `indice = nb_pts - 1 + days_ahead`. Or les prix WFP DataBridges sont
  typiquement **mensuels** (parfois hebdomadaires), pas journaliers.
- **Impact :** sur des données non journalières, (a) la période saisonnière
  est mal calée (365 *observations* ≠ 1 an) et (b) `days_ahead` est ajouté à
  un index d'observations, si bien que « prévoir à 7 jours » revient à
  avancer de 7 *pas* (≈ 7 mois si mensuel). La prévision et son intervalle
  peuvent être significativement biaisés.
- **Recommandation :** dériver les features du **temps réel** (jour de
  l'année à partir de la colonne `date`) et convertir `days_ahead` en
  position temporelle réelle plutôt qu'en nombre d'observations.

### 🟡 3.4 — Clés de configuration mortes

Plusieurs entrées de `config.py` ne sont jamais lues :

| Clé | Ligne | État |
|---|---|---|
| `MODELS_DIR` (`_ml/models`) | `config.py:33` | Dossier `kadi/_ml/` **inexistant**, clé non utilisée |
| `CACHE_DB_BACKUP` | `config.py:20` | Jamais référencée |
| `EXCHANGE_RATES` | `config.py:212` | Jamais lue (cf. 3.1) |
| `consommation_l_per_100km` | `config.py:135` | Jamais lue (cf. 3.2) |
| `min_history_weeks` | `config.py:83` | Jamais lue |

**Recommandation :** supprimer ces clés ou les câbler. Elles laissent croire
à des comportements configurables qui n'existent pas.

### 🟡 3.5 — Bornes GPS incohérentes entre modules

- **Où :** `market/__init__.py:22-25` code en dur `_LAT_MIN=6.0…` au lieu de
  lire `CONFIG`, alors que `config.py:66-71` définit
  `weather.gps_validation_bbox` (lat 2.5–12.5) et `config.py:186-191` une
  bbox kidas encore différente (Afrique de l'Ouest élargie).
- **Impact :** un même point peut être **accepté par un module et rejeté par
  un autre**. Maintenance dispersée.
- **Recommandation :** centraliser une bbox « Bénin » unique dans `CONFIG` et
  la réutiliser partout.

### 🟡 3.6 — `requirements.txt` désynchronisé de `pyproject.toml`

- **Où :** `requirements.txt:22,68` déclare `xlrd<2.0` et `dask>=2023.1`,
  **absents** de `pyproject.toml`.
- **Impact :** `pip install kadipy` (qui lit `pyproject.toml`) n'installera ni
  `xlrd` (lecture `.xls`) ni `dask`. Divergence entre l'environnement de dev
  et l'installation finale.
- **Recommandation :** faire de `pyproject.toml` la source unique et
  régénérer / supprimer `requirements.txt`.

### 🟡 3.7 — Détails mineurs

- **`np.random.normal` sans graine** (`pricing.py:70`,
  `data_ingestion.py:641`) : les prix simulés varient à chaque appel
  (non reproductible). Impact faible car `is_simulated=True`, mais gêne les
  tests et la démonstration. → fixer un `seed` ou `np.random.default_rng`.
- **User-Agent Nominatim obsolète** (`logistics.py:316` : `"KadiPy/0.1.0"`)
  alors que la version est `1.0.0`. → dériver de `kadi.__version__`.
- **`EXCHANGE_RATES` statique** avec commentaire « Mise à jour quotidienne
  prévue » non implémenté (cf. 3.1).

---

## 4. Tests et CI

- **238 tests unitaires passent** (couverture des 3 modules, incluant
  performance et intégration mockée).
- **Tests d'intégration réseau** isolés via le marqueur `integration`
  (`pytest.ini`) — bonne pratique ; non exécutés hors ligne.
- **CI GitHub Actions solide** :
  - `tests.yml` : matrice Python 3.9→3.12 sur push/PR.
  - `publish.yml` : **PyPI via Trusted Publishing (OIDC)**, sans secret stocké.
  - `docs.yml` : MkDocs + `mike` (doc versionnée).
  - Permissions restreintes (`contents: read`).
- **Suggestions :** ajouter `pytest-cov` au workflow pour suivre la
  couverture ; ajouter un test de non-régression sur la formule de coût
  carburant (§3.2) une fois clarifiée.

---

## 5. Sécurité

Aucun problème identifié :

- Token WFP lu depuis l'environnement/`.env` et **jamais journalisé**
  (`data_ingestion.py:221-257`).
- Publication PyPI en OIDC (aucun token stocké).
- Pas d'`eval`/`exec`, pas de désérialisation non sûre.
- Aucun `except:` nu, aucun `print()` en bibliothèque, aucun `TODO/FIXME`.
- Permissions CI minimales.

---

## 6. Recommandations priorisées

| Priorité | Action | Réf. | Effort |
|---|---|---|---|
| 🟠 Haute | Unifier la source des taux de change (supprimer ou câbler `config.EXCHANGE_RATES`) | 3.1 | Faible |
| 🟠 Haute | Clarifier / corriger la formule de coût carburant (facteur conso) | 3.2 | Faible |
| 🟠 Haute | Baser les features de prévision sur le temps réel, pas l'index d'obs. | 3.3 | Moyen |
| 🟡 Moyenne | Supprimer les clés de config mortes | 3.4 | Faible |
| 🟡 Moyenne | Centraliser la bbox GPS « Bénin » dans `CONFIG` | 3.5 | Faible |
| 🟡 Moyenne | Réconcilier `requirements.txt` ↔ `pyproject.toml` | 3.6 | Faible |
| 🟡 Basse | Graine RNG, User-Agent versionné, `pytest-cov` | 3.7, §4 | Faible |

---

## 7. Conclusion

KadiPy reste un projet **bien conçu, bien documenté et bien testé**, sans bug
bloquant. Cette analyse approfondie confirme la qualité de l'architecture,
mais identifie un **noyau de dette de configuration et deux points de
modélisation** (taux de change fantômes, coût carburant sous-modélisé,
saisonnalité indexée sur les observations) qui méritent correction : ils
touchent directement la justesse des coûts logistiques et des prévisions de
prix, c'est-à-dire le cœur de la valeur métier du module `market`. Le
traitement des trois recommandations « Haute » ci-dessus fiabiliserait les
résultats sans remettre en cause l'existant.
