# Données d'Entrée du Solveur OptaPlanner

Ce document décrit **toutes les données** que le solveur reçoit en entrée pour résoudre le problème de planification.

---

# 1. VUE D'ENSEMBLE

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INPUT DU SOLVEUR                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. PARAMÈTRES GLOBAUX                                               │
│     └── Semaine à planifier (date_debut, date_fin)                   │
│                                                                      │
│  2. DONNÉES DE RÉFÉRENCE (statiques)                                 │
│     ├── Sites                                                        │
│     ├── Emplacements                                                 │
│     ├── Spécialités                                                  │
│     └── Skills                                                       │
│                                                                      │
│  3. RESSOURCES (secrétaires)                                         │
│     ├── Type 1 : Horaire Flexible                                    │
│     │   └── nombre_jours_semaine + absences                          │
│     └── Type 2 : Horaire Fixe                                        │
│         └── shifts_fixes (créneaux prédéfinis)                       │
│                                                                      │
│  4. DEMANDE (shifts à couvrir)                                       │
│     ├── Besoins par emplacement/période                              │
│     └── Rôles de fermeture requis                                    │
│                                                                      │
│  5. PRÉFÉRENCES                                                      │
│     ├── Préférences de site par ressource                            │
│     ├── Préférences de skill par ressource                           │
│     └── Préférences de médecin par ressource                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

# 2. PARAMÈTRES GLOBAUX

```json
{
  "planning_request": {
    "date_debut": "2024-01-15",      // Lundi de la semaine
    "date_fin": "2024-01-20",        // Samedi de la semaine
    "solver_timeout_ms": 300000      // 5 minutes max
  }
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `date_debut` | Date (ISO 8601) | Premier jour de la semaine (lundi) |
| `date_fin` | Date (ISO 8601) | Dernier jour de la semaine (samedi) |
| `solver_timeout_ms` | Entier | Temps max de résolution en ms |

---

# 3. DONNÉES DE RÉFÉRENCE

## 3.1 Sites

Les **sites** sont les lieux géographiques. Les ressources ont des **préférences de site**.

```json
{
  "sites": [
    {
      "id": "site-la-vallee",
      "nom": "Clinique La Vallée",
      "adresse": "Rue de Chaux 4, 2800 Delémont",
      "distance_type": "reference"
    },
    {
      "id": "site-porrentruy",
      "nom": "Porrentruy",
      "adresse": "Fbg Saint-Germain 2, 2900 Porrentruy",
      "distance_type": "distant"
    },
    {
      "id": "site-vieille-ville",
      "nom": "Vieille Ville Delémont",
      "adresse": "Place de la Liberté 2, 2800 Delémont",
      "distance_type": "proche"
    }
  ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique |
| `nom` | String | Nom du site |
| `adresse` | String | Adresse physique |
| `distance_type` | Enum | `reference`, `proche`, `distant` |

## 3.2 Emplacements

Les **emplacements** sont les services/départements dans un site.

```json
{
  "emplacements": [
    {
      "id": "7c8abe96-0a6b-44eb-857f-ad69036ebc88",
      "nom": "Ophtalmologie",
      "site_id": "site-la-vallee",
      "type": "consultation",
      "specialite_id": "spec-ophtalmo",
      "fermeture": true
    },
    {
      "id": "86f1047f-c4ff-441f-a064-42ee2f8ef37a",
      "nom": "Bloc opératoire",
      "site_id": "site-la-vallee",
      "type": "bloc",
      "specialite_id": null,
      "fermeture": false
    },
    {
      "id": "00000000-0000-0000-0000-000000000001",
      "nom": "Administratif",
      "site_id": null,
      "type": "admin",
      "specialite_id": null,
      "fermeture": false
    }
  ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique |
| `nom` | String | Nom de l'emplacement |
| `site_id` | UUID \| null | Site parent (null pour Admin) |
| `type` | Enum | `consultation`, `bloc`, `admin` |
| `specialite_id` | UUID \| null | Spécialité liée (consultations) |
| `fermeture` | Boolean | Nécessite des rôles 1R/2F |

## 3.3 Spécialités

```json
{
  "specialites": [
    {
      "id": "spec-ophtalmo",
      "nom": "Ophtalmologie",
      "type_staffing": "B",
      "skills_requis": ["skill-accueil-ophtalmo"]
    },
    {
      "id": "spec-dermato",
      "nom": "Dermatologie",
      "type_staffing": "A",
      "skills_requis": ["skill-aide-dermato", "skill-accueil-dermato"],
      "ratios": {
        "1_medecin": { "aide": 2, "accueil": 1 },
        "2_medecins_plus": { "aide": 2, "accueil": 2 }
      }
    },
    {
      "id": "spec-gastro",
      "nom": "Gastroentérologie",
      "type_staffing": "A",
      "skills_requis": ["skill-aide-gastro", "skill-accueil-gastro"],
      "ratios": {
        "default": { "aide": 2, "accueil": 1 }
      }
    }
  ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique |
| `nom` | String | Nom de la spécialité |
| `type_staffing` | Enum | `A` (ratio fixe), `B` (simple), `C` (bloc), `D` (admin) |
| `skills_requis` | Array[UUID] | Skills nécessaires |
| `ratios` | Object | Ratios par nombre de médecins (Type A) |

## 3.4 Skills

```json
{
  "skills": [
    {
      "id": "skill-accueil-ophtalmo",
      "nom": "Accueil Ophtalmo",
      "type": "accueil",
      "specialite_id": "spec-ophtalmo"
    },
    {
      "id": "skill-aide-dermato",
      "nom": "Aide de salle Dermato",
      "type": "aide_salle",
      "specialite_id": "spec-dermato"
    },
    {
      "id": "skill-1r",
      "nom": "1R - Première Responsabilité",
      "type": "fermeture",
      "specialite_id": null
    },
    {
      "id": "skill-2f",
      "nom": "2F - Deuxième Fermeture",
      "type": "fermeture",
      "specialite_id": null
    }
  ]
}
```

---

# 4. RESSOURCES (Secrétaires)

## 4.1 Structure Commune

```json
{
  "ressources": [
    {
      "id": "1e5339aa-5e82-4295-b918-e15a580b3396",
      "nom": "Bron",
      "prenom": "Florence",
      "active": true,
      "horaire_flexible": true,
      "prefered_admin": false,
      "nombre_demi_journees_admin": 2
    }
  ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique |
| `nom`, `prenom` | String | Identification |
| `active` | Boolean | Disponible pour planification |
| `horaire_flexible` | Boolean | **Détermine le type** (voir ci-dessous) |
| `prefered_admin` | Boolean | Préfère l'administratif |
| `nombre_demi_journees_admin` | Integer | Objectif shifts admin/semaine |

## 4.2 Type 1 : Horaire Flexible (`horaire_flexible = true`)

Le solveur décide **quels jours** assigner ET **où**.

```json
{
  "ressource_flexible": {
    "id": "res-marie",
    "horaire_flexible": true,

    // INPUT SPÉCIFIQUE TYPE 1
    "nombre_jours_semaine": 4,     // ← EXACT, pas un %
    "absences": [
      {
        "date": "2024-01-17",
        "periode": null            // null = jour complet
      },
      {
        "date": "2024-01-19",
        "periode": "matin"         // demi-journée spécifique
      }
    ]
  }
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `nombre_jours_semaine` | Integer | Nombre **EXACT** de jours à assigner (contrainte HARD) |
| `absences` | Array | Créneaux où la ressource est indisponible |
| `absences[].date` | Date | Date de l'absence |
| `absences[].periode` | Enum \| null | `matin`, `apres_midi`, ou `null` (jour complet) |

**Contrainte HARD** : Le solveur DOIT assigner exactement `nombre_jours_semaine` jours.

**Exemple** :
- Marie : `nombre_jours_semaine = 4`, absence mercredi
- Jours disponibles : {lundi, mardi, jeudi, vendredi, samedi} = 5 jours
- Le solveur choisit 4 jours parmi ces 5

## 4.3 Type 2 : Horaire Fixe (`horaire_flexible = false`)

Les créneaux sont **prédéfinis**. Le solveur décide uniquement **où** assigner.

```json
{
  "ressource_fixe": {
    "id": "res-pierre",
    "horaire_flexible": false,

    // INPUT SPÉCIFIQUE TYPE 2
    "shifts_fixes": [
      { "date": "2024-01-15", "periode": "matin" },
      { "date": "2024-01-15", "periode": "apres_midi" },
      { "date": "2024-01-16", "periode": "matin" },
      { "date": "2024-01-18", "periode": "apres_midi" }
    ]
  }
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `shifts_fixes` | Array | Liste des créneaux où la ressource DOIT travailler |
| `shifts_fixes[].date` | Date | Date du shift |
| `shifts_fixes[].periode` | Enum | `matin` ou `apres_midi` |

**Pas de contrainte de nombre** : Le solveur assigne chaque shift de la liste.

**Exemple** :
- Pierre : 4 shifts fixes définis
- Le solveur DOIT assigner Pierre sur ces 4 créneaux
- Il choisit l'emplacement pour chaque créneau

## 4.4 Tableau Comparatif

| Aspect | Type 1 (Flexible) | Type 2 (Fixe) |
|--------|-------------------|---------------|
| **Input principal** | `nombre_jours_semaine` + `absences` | `shifts_fixes` |
| **Solveur décide** | Quels jours + où | Seulement où |
| **Contrainte HARD** | Exactement N jours | Tous les shifts de la liste |
| **Absences** | Bloquent des jours | N/A (pas de shifts ces jours) |

---

# 5. DEMANDE (Shifts à Couvrir)

## 5.1 Besoins par Emplacement

Générés à partir des médecins présents.

```json
{
  "besoins": [
    {
      "id": "besoin-001",
      "date": "2024-01-15",
      "periode": "matin",
      "emplacement_id": "7c8abe96-0a6b-44eb-857f-ad69036ebc88",
      "skills_demandes": [
        { "skill_id": "skill-accueil-ophtalmo", "quantite": 3 }
      ],
      "medecins_presents": [
        { "id": "med-001", "nom": "Dr. Martin", "besoin_secretaires": 1.2 },
        { "id": "med-002", "nom": "Dr. Dupont", "besoin_secretaires": 1.2 }
      ]
    },
    {
      "id": "besoin-002",
      "date": "2024-01-15",
      "periode": "matin",
      "emplacement_id": "d82c55ee-2964-49d4-a578-417b55b557ec",
      "skills_demandes": [
        { "skill_id": "skill-aide-dermato", "quantite": 2 },
        { "skill_id": "skill-accueil-dermato", "quantite": 2 }
      ],
      "medecins_presents": [
        { "id": "med-003", "nom": "Dr. Bernard", "besoin_secretaires": 1.2 },
        { "id": "med-004", "nom": "Dr. Thomas", "besoin_secretaires": 1.2 }
      ]
    }
  ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique du besoin |
| `date` | Date | Date du shift |
| `periode` | Enum | `matin` ou `apres_midi` |
| `emplacement_id` | UUID | Emplacement concerné |
| `skills_demandes` | Array | Skills requis avec quantité |
| `medecins_presents` | Array | Médecins présents (pour info) |

## 5.2 Besoins de Fermeture

Pour les emplacements avec `fermeture = true`.

```json
{
  "besoins_fermeture": [
    {
      "id": "fermeture-001",
      "date": "2024-01-15",
      "emplacement_id": "7c8abe96-0a6b-44eb-857f-ad69036ebc88",
      "roles_requis": ["1R", "2F"],
      "journee_complete": true     // Médecins matin ET après-midi
    },
    {
      "id": "fermeture-002",
      "date": "2024-01-16",
      "emplacement_id": "043899a1-a232-4c4b-9d7d-0eb44dad00ad",
      "roles_requis": ["1R", "2F"],
      "journee_complete": false,   // Médecins matin uniquement
      "periode_active": "matin"
    }
  ]
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `journee_complete` | Boolean | Médecins présents matin ET PM |
| `periode_active` | Enum \| null | Si `journee_complete = false`, quelle période |

**Règle importante** : Si `journee_complete = true`, les rôles 1R et 2F doivent être assignés à la **même personne toute la journée**.

---

# 6. PRÉFÉRENCES

## 6.1 Préférences de Site

```json
{
  "preferences_site": [
    {
      "ressource_id": "res-marie",
      "site_id": "site-la-vallee",
      "priorite": 1                // P1 = préféré
    },
    {
      "ressource_id": "res-marie",
      "site_id": "site-vieille-ville",
      "priorite": 2
    },
    {
      "ressource_id": "res-marie",
      "site_id": "site-porrentruy",
      "priorite": 3                // P3 = moins préféré
    }
  ]
}
```

| Priorité | Score Soft |
|----------|------------|
| P1 | +40 |
| P2 | +35 |
| P3 | +30 |
| P4 | +25 |

## 6.2 Préférences de Skill

Définit aussi l'**éligibilité** : sans préférence de skill = pas éligible.

```json
{
  "preferences_skill": [
    {
      "ressource_id": "res-marie",
      "skill_id": "skill-accueil-ophtalmo",
      "priorite": 1
    },
    {
      "ressource_id": "res-marie",
      "skill_id": "skill-accueil-dermato",
      "priorite": 2
    }
  ]
}
```

| Priorité | Score Soft | Éligibilité |
|----------|------------|-------------|
| P1 | +100 | ✓ Éligible |
| P2 | +80 | ✓ Éligible |
| P3 | +60 | ✓ Éligible |
| Aucune | - | ✗ Non éligible |

## 6.3 Préférences de Médecin

```json
{
  "preferences_medecin": [
    {
      "ressource_id": "res-sara",
      "medecin_id": "fda323f4-3efd-4c78-8b63-7d660fcd7eea",
      "priorite": 1
    }
  ]
}
```

| Priorité | Score Soft |
|----------|------------|
| P1 | +70 |
| P2 | +50 |

---

# 7. RÉSUMÉ DES INPUTS

## 7.1 Fichiers/Tables Requis

| Donnée | Source | Fréquence MAJ |
|--------|--------|---------------|
| Sites | Table `sites` | Rare |
| Emplacements | Table `emplacements` | Rare |
| Spécialités | Table `specialites` | Rare |
| Skills | Table `skills` | Rare |
| Ressources | Table `secretaires` | Hebdomadaire |
| Absences | Table `absences` | Hebdomadaire |
| Shifts fixes | Table `horaires_fixes` | Hebdomadaire |
| Besoins | Calculé depuis `plannings_medecins` | Hebdomadaire |
| Préférences | Tables `preferences_*` | Occasionnel |

## 7.2 Validation des Inputs

Avant d'appeler le solveur, valider :

| Validation | Erreur si violation |
|------------|---------------------|
| `nombre_jours_semaine` ≤ jours disponibles | Impossible à résoudre |
| Chaque `shifts_fixes` référence une date valide | Donnée incohérente |
| Chaque préférence référence des IDs existants | Référence invalide |
| Au moins 1 ressource éligible par skill demandé | Besoin non couvrable |

---

# 8. EXEMPLE COMPLET D'INPUT

```json
{
  "planning_request": {
    "date_debut": "2024-01-15",
    "date_fin": "2024-01-20",
    "solver_timeout_ms": 300000
  },

  "sites": [...],
  "emplacements": [...],
  "specialites": [...],
  "skills": [...],

  "ressources": [
    {
      "id": "res-marie",
      "nom": "Dupont",
      "prenom": "Marie",
      "horaire_flexible": true,
      "nombre_jours_semaine": 4,
      "absences": [
        { "date": "2024-01-17", "periode": null }
      ],
      "prefered_admin": false
    },
    {
      "id": "res-pierre",
      "nom": "Martin",
      "prenom": "Pierre",
      "horaire_flexible": false,
      "shifts_fixes": [
        { "date": "2024-01-15", "periode": "matin" },
        { "date": "2024-01-15", "periode": "apres_midi" },
        { "date": "2024-01-16", "periode": "matin" }
      ],
      "prefered_admin": true
    }
  ],

  "besoins": [...],
  "besoins_fermeture": [...],

  "preferences_site": [...],
  "preferences_skill": [...],
  "preferences_medecin": [...]
}
```

---

*Document de spécification des données d'entrée OptaPlanner*
*Voir aussi : OUTPUT_SCHEMA.md, CONSTRAINTS.md*
