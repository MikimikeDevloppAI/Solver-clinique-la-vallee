# Données de Sortie du Solveur OptaPlanner

Ce document décrit **toutes les données** que le solveur retourne après résolution.

---

# 1. VUE D'ENSEMBLE

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OUTPUT DU SOLVEUR                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. MÉTADONNÉES DE RÉSOLUTION                                        │
│     ├── Statut (OPTIMAL, FEASIBLE, INFEASIBLE)                       │
│     ├── Score final (Hard/Medium/Soft)                               │
│     └── Temps de résolution                                          │
│                                                                      │
│  2. ASSIGNATIONS                                                     │
│     └── Liste de (ressource, date, période, emplacement, skill)      │
│                                                                      │
│  3. RÔLES DE FERMETURE                                               │
│     └── Liste de (ressource, date, emplacement, rôle)                │
│                                                                      │
│  4. ANALYSE DE QUALITÉ (optionnel)                                   │
│     ├── Contraintes violées                                          │
│     ├── Score par catégorie                                          │
│     └── Besoins non couverts                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

# 2. MÉTADONNÉES DE RÉSOLUTION

```json
{
  "resolution": {
    "statut": "FEASIBLE",
    "score": {
      "hard": 0,
      "medium": 0,
      "soft": -1250
    },
    "temps_resolution_ms": 180000,
    "iterations": 125000,
    "timestamp": "2024-01-14T10:30:00Z"
  }
}
```

## 2.1 Statuts Possibles

| Statut | Signification | Score Hard | Score Medium |
|--------|---------------|------------|--------------|
| `OPTIMAL` | Meilleure solution trouvée | 0 | 0 |
| `FEASIBLE` | Solution valide trouvée | 0 | 0 |
| `INFEASIBLE` | Aucune solution valide | < 0 | - |
| `NO_SOLUTION` | Pas de solution (timeout) | - | - |

## 2.2 Interprétation du Score

```
Score = HardMediumSoftScore(hard, medium, soft)

Comparaison lexicographique :
  (0, 0, -100) > (-1, 0, +1000)    // Hard prime
  (0, 0, -100) > (0, -1, +1000)    // Medium prime sur Soft
  (0, 0, -100) < (0, 0, -50)       // Plus le soft est proche de 0, mieux c'est
```

| Niveau | Objectif | Valeur idéale |
|--------|----------|---------------|
| Hard | 0 | **Obligatoire** pour solution valide |
| Medium | 0 | **Souhaitable** (tous besoins couverts) |
| Soft | Max | Plus élevé = meilleure qualité |

---

# 3. ASSIGNATIONS

## 3.1 Structure d'une Assignation

```json
{
  "assignations": [
    {
      "id": "assign-001",
      "ressource_id": "res-marie",
      "date": "2024-01-15",
      "periode": "matin",
      "emplacement_id": "7c8abe96-0a6b-44eb-857f-ad69036ebc88",
      "skill_id": "skill-accueil-ophtalmo",
      "score_contribution": {
        "skill_preference": 100,
        "site_preference": 40,
        "medecin_preference": 0,
        "total": 100
      }
    }
  ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique de l'assignation |
| `ressource_id` | UUID | Secrétaire assignée |
| `date` | Date | Date du shift |
| `periode` | Enum | `matin` ou `apres_midi` |
| `emplacement_id` | UUID | Emplacement assigné |
| `skill_id` | UUID | Skill utilisé pour cette assignation |
| `score_contribution` | Object | Détail du score (optionnel) |

## 3.2 Assignation Administrative

Pour l'emplacement Admin (pas de skill spécifique) :

```json
{
  "id": "assign-002",
  "ressource_id": "res-pierre",
  "date": "2024-01-15",
  "periode": "apres_midi",
  "emplacement_id": "00000000-0000-0000-0000-000000000001",
  "skill_id": null,
  "is_admin": true
}
```

## 3.3 Exemple de Planning Complet

```json
{
  "assignations": [
    // Lundi matin - Ophtalmo
    { "ressource_id": "res-marie", "date": "2024-01-15", "periode": "matin",
      "emplacement_id": "ophtalmo-la-vallee", "skill_id": "skill-accueil-ophtalmo" },
    { "ressource_id": "res-anna", "date": "2024-01-15", "periode": "matin",
      "emplacement_id": "ophtalmo-la-vallee", "skill_id": "skill-accueil-ophtalmo" },
    { "ressource_id": "res-luc", "date": "2024-01-15", "periode": "matin",
      "emplacement_id": "ophtalmo-la-vallee", "skill_id": "skill-accueil-ophtalmo" },

    // Lundi matin - Dermato
    { "ressource_id": "res-sophie", "date": "2024-01-15", "periode": "matin",
      "emplacement_id": "dermato-la-vallee", "skill_id": "skill-aide-dermato" },
    { "ressource_id": "res-jean", "date": "2024-01-15", "periode": "matin",
      "emplacement_id": "dermato-la-vallee", "skill_id": "skill-aide-dermato" },
    { "ressource_id": "res-claire", "date": "2024-01-15", "periode": "matin",
      "emplacement_id": "dermato-la-vallee", "skill_id": "skill-accueil-dermato" },
    { "ressource_id": "res-paul", "date": "2024-01-15", "periode": "matin",
      "emplacement_id": "dermato-la-vallee", "skill_id": "skill-accueil-dermato" },

    // Lundi matin - Admin
    { "ressource_id": "res-pierre", "date": "2024-01-15", "periode": "matin",
      "emplacement_id": "admin", "skill_id": null, "is_admin": true }
  ]
}
```

---

# 4. RÔLES DE FERMETURE

## 4.1 Structure

```json
{
  "roles_fermeture": [
    {
      "id": "fermeture-assign-001",
      "ressource_id": "res-marie",
      "date": "2024-01-15",
      "emplacement_id": "7c8abe96-0a6b-44eb-857f-ad69036ebc88",
      "role": "1R",
      "journee_complete": true
    },
    {
      "id": "fermeture-assign-002",
      "ressource_id": "res-anna",
      "date": "2024-01-15",
      "emplacement_id": "7c8abe96-0a6b-44eb-857f-ad69036ebc88",
      "role": "2F",
      "journee_complete": true
    }
  ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `ressource_id` | UUID | Secrétaire assignée au rôle |
| `date` | Date | Date du rôle |
| `emplacement_id` | UUID | Emplacement avec fermeture |
| `role` | Enum | `1R`, `2F`, ou `3F` |
| `journee_complete` | Boolean | True si même personne matin + PM |

## 4.2 Règle de Cohérence

Si `journee_complete = true`, la ressource avec rôle 1R ou 2F **doit** apparaître dans les assignations pour les **deux périodes** (matin ET après-midi) sur le même emplacement.

---

# 5. ANALYSE DE QUALITÉ

## 5.1 Résumé par Ressource

```json
{
  "analyse_ressources": [
    {
      "ressource_id": "res-marie",
      "nom_complet": "Marie Dupont",
      "jours_assignes": 4,
      "jours_requis": 4,
      "shifts_total": 8,
      "repartition": {
        "ophtalmo-la-vallee": 4,
        "dermato-la-vallee": 2,
        "admin": 2
      },
      "roles_fermeture": {
        "1R": 1,
        "2F": 0
      },
      "score_fermeture": 10,
      "jours_site_distant": 0,
      "score_preference_moyen": 85
    }
  ]
}
```

## 5.2 Contraintes Violées (si score < 0)

```json
{
  "violations": {
    "hard": [],
    "medium": [
      {
        "type": "SKILL_MANQUANT",
        "besoin_id": "besoin-042",
        "details": "Dermato mardi PM : 1 Accueil Dermato manquant",
        "penalite": -1000
      }
    ],
    "soft": [
      {
        "type": "P234_JOUR_3",
        "ressource_id": "res-luc",
        "details": "3ème jour à Porrentruy",
        "penalite": -50
      }
    ]
  }
}
```

## 5.3 Besoins Non Couverts

```json
{
  "besoins_non_couverts": [
    {
      "besoin_id": "besoin-042",
      "date": "2024-01-16",
      "periode": "apres_midi",
      "emplacement": "Dermatologie",
      "skill_manquant": "Accueil Dermato",
      "quantite_manquante": 1
    }
  ]
}
```

---

# 6. STATISTIQUES GLOBALES

```json
{
  "statistiques": {
    "total_assignations": 120,
    "total_ressources_utilisees": 15,
    "couverture": {
      "besoins_totaux": 48,
      "besoins_couverts": 47,
      "taux_couverture": 97.9
    },
    "fermeture": {
      "sites_avec_fermeture": 2,
      "roles_assignes": 20,
      "roles_manquants": 0
    },
    "qualite": {
      "score_preference_moyen": 78.5,
      "changements_site": 12,
      "ressources_site_distant": 3
    }
  }
}
```

---

# 7. FORMAT DE SORTIE COMPLET

```json
{
  "resolution": {
    "statut": "FEASIBLE",
    "score": { "hard": 0, "medium": 0, "soft": -1250 },
    "temps_resolution_ms": 180000,
    "timestamp": "2024-01-14T10:30:00Z"
  },

  "assignations": [
    {
      "ressource_id": "res-marie",
      "date": "2024-01-15",
      "periode": "matin",
      "emplacement_id": "ophtalmo-la-vallee",
      "skill_id": "skill-accueil-ophtalmo"
    },
    // ... autres assignations
  ],

  "roles_fermeture": [
    {
      "ressource_id": "res-marie",
      "date": "2024-01-15",
      "emplacement_id": "ophtalmo-la-vallee",
      "role": "1R",
      "journee_complete": true
    },
    // ... autres rôles
  ],

  "analyse_ressources": [...],
  "violations": {...},
  "besoins_non_couverts": [...],
  "statistiques": {...}
}
```

---

# 8. TRANSFORMATION POUR AFFICHAGE

## 8.1 Vue Planning Hebdomadaire par Ressource

```
┌──────────────┬─────────────────┬─────────────────┬─────────────────┐
│              │     LUNDI       │     MARDI       │    MERCREDI     │
│   Ressource  ├────────┬────────┼────────┬────────┼────────┬────────┤
│              │ Matin  │   PM   │ Matin  │   PM   │ Matin  │   PM   │
├──────────────┼────────┼────────┼────────┼────────┼────────┼────────┤
│ Marie Dupont │ Ophtal │ Ophtal │ Dermat │ Admin  │   -    │   -    │
│              │  (1R)  │  (1R)  │        │        │        │        │
├──────────────┼────────┼────────┼────────┼────────┼────────┼────────┤
│ Pierre Martin│ Admin  │ Dermat │ Ophtal │ Ophtal │ Porren │ Porren │
│              │        │        │  (2F)  │  (2F)  │        │        │
└──────────────┴────────┴────────┴────────┴────────┴────────┴────────┘
```

## 8.2 Vue Planning par Emplacement

```
OPHTALMOLOGIE - Lundi 15 janvier

  MATIN                          APRÈS-MIDI
  ──────────────────────────     ──────────────────────────
  • Marie Dupont (1R)            • Marie Dupont (1R)
  • Anna Leroy (2F)              • Anna Leroy (2F)
  • Luc Bernard                   • Luc Bernard

  Médecins: Dr. Martin, Dr. Dupont
```

---

# 9. CODES D'ERREUR

| Code | Signification | Action |
|------|---------------|--------|
| `INFEASIBLE_NO_RESOURCE` | Pas assez de ressources | Vérifier disponibilités |
| `INFEASIBLE_CONFLICT` | Conflits insolubles | Vérifier absences/fixes |
| `TIMEOUT_NO_SOLUTION` | Temps écoulé sans solution | Augmenter timeout |
| `INVALID_INPUT` | Données d'entrée invalides | Vérifier format |

---

*Document de spécification des données de sortie OptaPlanner*
*Voir aussi : INPUT_SCHEMA.md, CONSTRAINTS.md*
