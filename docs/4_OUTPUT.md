# Format de Sortie du Solveur

Ce document décrit le format des données générées par le solveur OptaPlanner.

---

## Vue d'Ensemble

Le solveur génère des entrées dans la table **planning** avec `type = 'secretaire'`.

```
ENTRÉE                              SORTIE
========                            ======
planning (type='medecin')    →      planning (type='secretaire')
besoins calculés             →      Assignations optimisées
```

---

# 1. STRUCTURE DE SORTIE

## Table planning (type = 'secretaire')

Chaque assignation est une entrée dans la table planning :

| Champ | Type | Description |
|-------|------|-------------|
| id | UUID | Identifiant unique |
| date | DATE | Date de l'assignation |
| demi_journee | ENUM | `matin` ou `apres_midi` |
| type | STRING | **`secretaire`** (valeur fixe) |
| personne_id | UUID | ID de la secrétaire assignée |
| site_id | UUID | ID du site/emplacement |
| skill_id | UUID | Skill utilisé (consultations) |
| operation_id | UUID | Opération couverte (bloc) |
| besoin_operation_id | UUID | Rôle au bloc (Instrumentiste, etc.) |
| is_1r | BOOLEAN | Rôle Première Responsabilité |
| is_2f | BOOLEAN | Rôle Deuxième Fermeture |
| is_3f | BOOLEAN | Rôle Troisième Fermeture |

---

# 2. TYPES D'ASSIGNATIONS

## 2.1 Consultation

```json
{
  "type": "secretaire",
  "personne_id": "uuid-secretaire",
  "date": "2024-01-15",
  "demi_journee": "matin",
  "site_id": "d82c55ee-2964-49d4-a578-417b55b557ec",
  "skill_id": "skill-accueil-dermato",
  "operation_id": null,
  "besoin_operation_id": null,
  "is_1r": false,
  "is_2f": false,
  "is_3f": false
}
```

## 2.2 Consultation avec rôle fermeture

```json
{
  "type": "secretaire",
  "personne_id": "uuid-secretaire",
  "date": "2024-01-15",
  "demi_journee": "matin",
  "site_id": "7c8abe96-0a6b-44eb-857f-ad69036ebc88",
  "skill_id": "skill-accueil-ophtalmo",
  "operation_id": null,
  "besoin_operation_id": null,
  "is_1r": true,
  "is_2f": false,
  "is_3f": false
}
```

## 2.3 Bloc opératoire

```json
{
  "type": "secretaire",
  "personne_id": "uuid-secretaire",
  "date": "2024-01-15",
  "demi_journee": "matin",
  "site_id": "86f1047f-c4ff-441f-a064-42ee2f8ef37a",
  "skill_id": null,
  "operation_id": "uuid-operation",
  "besoin_operation_id": "besoin-instrumentiste",
  "is_1r": false,
  "is_2f": false,
  "is_3f": false
}
```

## 2.4 Administratif

```json
{
  "type": "secretaire",
  "personne_id": "uuid-secretaire",
  "date": "2024-01-15",
  "demi_journee": "apres_midi",
  "site_id": "00000000-0000-0000-0000-000000000001",
  "skill_id": null,
  "operation_id": null,
  "besoin_operation_id": null,
  "is_1r": false,
  "is_2f": false,
  "is_3f": false
}
```

---

# 3. CONTRAINTES SUR LA SORTIE

## Rôles de fermeture

Si un emplacement a `fermeture = true` et des médecins sont présents :

| Situation | Contrainte |
|-----------|-----------|
| Médecins matin ET après-midi | `is_1r` et `is_2f` doivent être sur la **même personne** matin ET après-midi |
| Médecins matin uniquement | `is_1r` et `is_2f` le matin, la personne peut être en Admin l'après-midi |
| Paul Jacquier jeudi+vendredi | Un `is_3f = true` doit exister |

## Horaires fixes

Pour les secrétaires avec `horaire_flexible = false` :
- Chaque shift prédéfini doit avoir une assignation correspondante
- Le solveur choisit le site, pas le créneau

## Horaires flexibles

Pour les secrétaires avec `horaire_flexible = true` :
- Le nombre de jours assignés doit être **exactement** `nombre_jours_semaine`
- Pas d'assignation sur les créneaux d'absence

---

# 4. SCORE DE LA SOLUTION

Le solveur retourne également le score de la solution :

```json
{
  "score": {
    "hard": 0,
    "medium": 0,
    "soft": 12450
  },
  "feasible": true,
  "assignations": [...]
}
```

| Champ | Description |
|-------|-------------|
| `hard` | Doit être = 0 pour une solution valide |
| `medium` | Doit être = 0 idéalement (tous les besoins couverts) |
| `soft` | Plus c'est élevé, meilleure est la qualité |
| `feasible` | `true` si `hard = 0` |

---

# 5. STATISTIQUES

Le solveur peut fournir des statistiques supplémentaires :

## Par secrétaire

| Statistique | Description |
|-------------|-------------|
| `jours_assignes` | Nombre de jours travaillés |
| `jours_1r` | Nombre de jours en 1R |
| `jours_2f` | Nombre de jours en 2F |
| `score_fermeture` | jours_1R × 10 + jours_2F × 12 |
| `jours_site_p234` | Jours sur sites P2/P3/P4 |

## Par besoin

| Statistique | Description |
|-------------|-------------|
| `besoin_requis` | Nombre demandé |
| `besoin_couvert` | Nombre assigné |
| `deficit` | besoin_requis - besoin_couvert |

---

# 6. EXEMPLE COMPLET

```json
{
  "score": {
    "hard": 0,
    "medium": 0,
    "soft": 15230
  },
  "feasible": true,
  "assignations": [
    {
      "type": "secretaire",
      "personne_id": "1e5339aa-5e82-4295-b918-e15a580b3396",
      "date": "2024-01-15",
      "demi_journee": "matin",
      "site_id": "7c8abe96-0a6b-44eb-857f-ad69036ebc88",
      "skill_id": "skill-accueil-ophtalmo",
      "is_1r": true,
      "is_2f": false
    },
    {
      "type": "secretaire",
      "personne_id": "1e5339aa-5e82-4295-b918-e15a580b3396",
      "date": "2024-01-15",
      "demi_journee": "apres_midi",
      "site_id": "7c8abe96-0a6b-44eb-857f-ad69036ebc88",
      "skill_id": "skill-accueil-ophtalmo",
      "is_1r": true,
      "is_2f": false
    },
    {
      "type": "secretaire",
      "personne_id": "5d3af9e3-674b-48d6-b54f-bd84c9eee670",
      "date": "2024-01-15",
      "demi_journee": "matin",
      "site_id": "86f1047f-c4ff-441f-a064-42ee2f8ef37a",
      "besoin_operation_id": "besoin-instrumentiste",
      "operation_id": "op-001"
    }
  ],
  "statistiques": {
    "secretaires": {
      "1e5339aa-...": {
        "jours_assignes": 4,
        "jours_1r": 2,
        "jours_2f": 0,
        "score_fermeture": 20
      }
    },
    "couverture": {
      "besoins_total": 45,
      "besoins_couverts": 45,
      "deficit": 0
    }
  }
}
```

---

*Voir aussi : 1_DOMAIN_MODEL.md, 2_NEEDS_CALCULATION.md, 3_CONSTRAINTS.md*
