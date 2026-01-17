# Schéma de Base de Données - Planification des Secrétaires Médicales

Ce document décrit **toutes les tables** et leurs relations pour le système de planification.

---

# 1. VUE D'ENSEMBLE

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          BASE DE DONNÉES                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  STRUCTURE GÉOGRAPHIQUE           RESSOURCES (Secrétaires)                   │
│  ========================         ========================                   │
│  sites                            secretaires                                │
│    └── emplacements                 ├── horaire (flexible/fixe)              │
│          └── specialites            ├── preferences_site                     │
│                └── skills           ├── preferences_skill                    │
│                                     ├── preferences_medecin                  │
│                                     ├── preferences_besoin_operation         │
│                                     └── absences / shifts_fixes              │
│                                                                              │
│  MÉDECINS                         BLOC OPÉRATOIRE                            │
│  ========                         ================                           │
│  medecins                         besoins_operations                         │
│    ├── besoin_secretaires         types_intervention                         │
│    └── specialite                   └── types_intervention_besoins_personnel │
│                                   planning_genere_bloc_operatoire            │
│                                                                              │
│  PLANNINGS                                                                   │
│  =========                                                                   │
│  planning_medecins (consultations)                                           │
│  planning_genere_bloc_operatoire (opérations)                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# 2. STRUCTURE GÉOGRAPHIQUE

## 2.1 Table `sites`

Les **sites** sont les lieux géographiques où travaillent les secrétaires.

```sql
CREATE TABLE sites (
    id UUID PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    adresse VARCHAR(255),
    distance_type VARCHAR(20)  -- 'reference', 'proche', 'distant'
);
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique |
| `nom` | String | Nom du site |
| `adresse` | String | Adresse physique |
| `distance_type` | Enum | `reference` (site principal), `proche`, `distant` |

**Données actuelles :**

| ID | Nom | Type |
|----|-----|------|
| `site-la-vallee` | Clinique La Vallée | reference |
| `site-porrentruy` | Porrentruy | distant |
| `site-vieille-ville` | Vieille Ville Delémont | proche |

---

## 2.2 Table `emplacements`

Les **emplacements** sont les services/départements dans un site.

```sql
CREATE TABLE emplacements (
    id UUID PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    site_id UUID REFERENCES sites(id),
    type VARCHAR(20) NOT NULL,       -- 'consultation', 'bloc', 'admin'
    specialite_id UUID REFERENCES specialites(id),
    fermeture BOOLEAN DEFAULT FALSE,
    actif BOOLEAN DEFAULT TRUE
);
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique |
| `nom` | String | Nom de l'emplacement |
| `site_id` | UUID | Site parent (null pour Admin) |
| `type` | Enum | `consultation`, `bloc`, `admin` |
| `specialite_id` | UUID | Spécialité liée (pour consultations) |
| `fermeture` | Boolean | Nécessite rôles 1R/2F si médecins présents |

**Données actuelles :**

| UUID | Nom | Site | Type | Fermeture |
|------|-----|------|------|-----------|
| `7c8abe96-...` | Ophtalmologie | La Vallée | consultation | ✓ |
| `d82c55ee-...` | Dermatologie | La Vallée | consultation | ✗ |
| `c3bd13...` | Angiologie | La Vallée | consultation | ✗ |
| `0e1b31...` | Rhumatologie | La Vallée | consultation | ✗ |
| `86f1047f-...` | Bloc opératoire | La Vallée | bloc | ✗ |
| `7723c334-...` | Gastroentérologie | Vieille Ville | consultation | ✗ |
| `043899a1-...` | Porrentruy | Porrentruy | consultation | ✓ |
| `00000000-...-000001` | Administratif | - | admin | ✗ |

---

## 2.3 Table `specialites`

Les **spécialités** définissent les règles de staffing.

```sql
CREATE TABLE specialites (
    id UUID PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    type_staffing CHAR(1) NOT NULL,  -- 'A', 'B', 'C', 'D'
    minimum_personnel INTEGER DEFAULT 0
);
```

**Types de staffing :**

| Type | Description | Emplacements |
|------|-------------|--------------|
| **A** | Ratio fixe Aide/Accueil | Gastro, Dermato |
| **B** | Skill unique, formule standard | Ophtalmo, Angio, Rhuma, ORL |
| **C** | Skills par intervention | Bloc opératoire |
| **D** | Aucun skill requis | Admin |

**Règles Type A (ratios fixes) :**

| Spécialité | Condition | Aide de salle | Accueil |
|------------|-----------|---------------|---------|
| Gastro | ≥1 médecin | 2 | 1 |
| Dermato | 1 médecin | 2 | 1 |
| Dermato | ≥2 médecins | 2 | 2 |

**Règles Type B (formule standard) :**

```
Demande = max(1, ceil(Σ besoin_secretaires de chaque médecin))
```

---

## 2.4 Table `skills`

Les **skills** sont les compétences requises pour chaque spécialité.

```sql
CREATE TABLE skills (
    id UUID PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    type VARCHAR(20) NOT NULL,  -- 'accueil', 'aide_salle', 'fermeture'
    specialite_id UUID REFERENCES specialites(id)
);
```

**Skills de Consultation :**

| Spécialité | Skill Aide de salle | Skill Accueil |
|------------|---------------------|---------------|
| Gastroentérologie | Aide de salle Gastro | Accueil Gastro |
| Dermatologie | Aide de salle Dermato | Accueil Dermato |
| Ophtalmologie | - | Accueil Ophtalmo |
| Angiologie | - | Accueil Angio |
| Rhumatologie | - | Accueil Rhuma |
| ORL | - | Accueil ORL |

**Skills de Fermeture :**

| Skill | Description |
|-------|-------------|
| 1R | Première Responsabilité |
| 2F | Deuxième Fermeture |
| 3F | Troisième Fermeture (cas Paul Jacquier) |

---

# 3. RESSOURCES (Secrétaires)

## 3.1 Table `secretaires`

```sql
CREATE TABLE secretaires (
    id UUID PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    active BOOLEAN DEFAULT TRUE,

    -- Type d'horaire
    horaire_flexible BOOLEAN NOT NULL,

    -- Si horaire_flexible = true
    nombre_jours_semaine INTEGER,

    -- Préférences admin
    prefered_admin BOOLEAN DEFAULT FALSE,
    nombre_demi_journees_admin INTEGER DEFAULT 0
);
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique |
| `nom`, `prenom` | String | Identification |
| `active` | Boolean | Disponible pour planification |
| `horaire_flexible` | Boolean | **Détermine le type d'horaire** |
| `nombre_jours_semaine` | Integer | Nombre EXACT de jours à assigner (si flexible) |
| `prefered_admin` | Boolean | Préfère l'administratif |
| `nombre_demi_journees_admin` | Integer | Objectif shifts admin/semaine |

### Deux types d'horaires :

| Type | `horaire_flexible` | Input | Solveur décide |
|------|-------------------|-------|----------------|
| **Flexible** | `true` | `nombre_jours_semaine` + absences | Quels jours + où |
| **Fixe** | `false` | `shifts_fixes` | Seulement où |

---

## 3.2 Table `absences` (pour horaire flexible)

```sql
CREATE TABLE absences (
    id UUID PRIMARY KEY,
    secretaire_id UUID REFERENCES secretaires(id),
    date DATE NOT NULL,
    periode VARCHAR(20)  -- 'matin', 'apres_midi', NULL = jour complet
);
```

**Exemple :**
```
Marie (flexible, 4 jours/semaine) :
  - Absence mercredi (jour complet)
  - Absence vendredi matin

→ Jours disponibles : lundi, mardi, jeudi, vendredi PM, samedi
→ Solveur choisit 4 jours parmi ceux-ci
```

---

## 3.3 Table `shifts_fixes` (pour horaire fixe)

```sql
CREATE TABLE shifts_fixes (
    id UUID PRIMARY KEY,
    secretaire_id UUID REFERENCES secretaires(id),
    date DATE NOT NULL,
    periode VARCHAR(20) NOT NULL  -- 'matin', 'apres_midi'
);
```

**Exemple :**
```
Pierre (fixe) :
  - Lundi matin
  - Lundi après-midi
  - Mardi matin
  - Jeudi après-midi

→ Solveur DOIT assigner Pierre sur ces 4 créneaux
→ Solveur choisit l'emplacement pour chaque créneau
```

---

## 3.4 Table `secretaires_preferences_site`

Préférences de site par secrétaire. **Sans préférence = non éligible pour ce site.**

```sql
CREATE TABLE secretaires_preferences_site (
    secretaire_id UUID REFERENCES secretaires(id),
    site_id UUID REFERENCES sites(id),
    priorite INTEGER NOT NULL,  -- 1 = P1 (préféré), 2 = P2, 3 = P3, 4 = P4
    PRIMARY KEY (secretaire_id, site_id)
);
```

| Priorité | Score Soft | Signification |
|----------|------------|---------------|
| P1 | +40 | Site préféré |
| P2 | +35 | Acceptable |
| P3 | +30 | Moins préféré |
| P4 | +25 | Dernier choix |
| Aucune | - | **Non éligible (contrainte HARD)** |

---

## 3.5 Table `secretaires_preferences_skill`

Préférences de skill par secrétaire. **Sans préférence = non éligible pour ce skill.**

```sql
CREATE TABLE secretaires_preferences_skill (
    secretaire_id UUID REFERENCES secretaires(id),
    skill_id UUID REFERENCES skills(id),
    priorite INTEGER NOT NULL,  -- 1 = P1, 2 = P2, 3 = P3
    PRIMARY KEY (secretaire_id, skill_id)
);
```

| Priorité | Score Soft | Éligibilité |
|----------|------------|-------------|
| P1 | +100 | ✓ Éligible |
| P2 | +80 | ✓ Éligible |
| P3 | +60 | ✓ Éligible |
| Aucune | - | ✗ **Non éligible (contrainte HARD)** |

---

## 3.6 Table `secretaires_preferences_medecin`

Préférences de médecin par secrétaire.

```sql
CREATE TABLE secretaires_preferences_medecin (
    secretaire_id UUID REFERENCES secretaires(id),
    medecin_id UUID REFERENCES medecins(id),
    priorite INTEGER NOT NULL,  -- 1 = P1, 2 = P2
    PRIMARY KEY (secretaire_id, medecin_id)
);
```

| Priorité | Score Soft |
|----------|------------|
| P1 | +70 |
| P2 | +50 |

---

## 3.7 Secrétaires Spéciales (Règles Individuelles)

| UUID | Nom | Règle |
|------|-----|-------|
| `1e5339aa-5e82-4295-b918-e15a580b3396` | Florence Bron | Pas 2F le mardi |
| `5d3af9e3-674b-48d6-b54f-bd84c9eee670` | Lucie Pratillo | Jamais 2F ni 3F |
| `121dc7d9-99dc-46bd-9b6c-d240ac6dc6c8` | Paul Jacquier | Déclenche 3F si présent jeudi+vendredi |
| `68e74e31-12a7-4fd3-836d-41e8abf57792` | Sara Bortolon | Bonus Dr. fda323f4 |
| `324639fa-2e3d-4903-a143-323a17b0d988` | Mirlinda Hasani | Bonus Dr. fda323f4 |

---

# 4. MÉDECINS

## 4.1 Table `medecins`

```sql
CREATE TABLE medecins (
    id UUID PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    specialite_id UUID REFERENCES specialites(id),
    besoin_secretaires DECIMAL(3,2) DEFAULT 1.2,
    is_obstetricienne BOOLEAN DEFAULT FALSE,
    actif BOOLEAN DEFAULT TRUE
);
```

| Champ | Type | Description |
|-------|------|-------------|
| `id` | UUID | Identifiant unique |
| `nom`, `prenom` | String | Identification |
| `specialite_id` | UUID | Spécialité du médecin |
| `besoin_secretaires` | Decimal | Coefficient de besoin (ex: 1.2) |
| `is_obstetricienne` | Boolean | Exception : `besoin_secretaires = 0` |

**Règle Obstétricienne :**
- Les obstétriciennes ne comptent PAS dans le calcul du besoin en secrétaires
- SAUF si site Ophtalmo avec uniquement des obstétriciennes → demande = 1 Accueil Ophtalmo

---

## 4.2 Table `planning_medecins` (Consultations)

Planning des médecins pour les consultations.

```sql
CREATE TABLE planning_medecins (
    id UUID PRIMARY KEY,
    medecin_id UUID REFERENCES medecins(id),
    emplacement_id UUID REFERENCES emplacements(id),
    date DATE NOT NULL,
    periode VARCHAR(20) NOT NULL,  -- 'matin', 'apres_midi'
    validated BOOLEAN DEFAULT FALSE
);
```

**Calcul de la demande :**

```
Pour chaque (emplacement, date, période) :

1. Lister les médecins présents
2. Exclure les obstétriciennes du calcul
3. Si 0 médecin restant → Demande = 0
4. Sinon :
   - Type A (Gastro, Dermato) : Appliquer ratios fixes
   - Type B (autres) : max(1, ceil(Σ besoin_secretaires))
5. Exception samedi : Demande = N_médecins (pas de ceiling)
```

---

# 5. BLOC OPÉRATOIRE

## 5.1 Table `besoins_operations`

Les **besoins d'opération** sont les rôles requis au bloc (équivalent des skills).

```sql
CREATE TABLE besoins_operations (
    id UUID PRIMARY KEY,
    code VARCHAR(20) NOT NULL,
    nom VARCHAR(100) NOT NULL,
    categorie VARCHAR(50),
    actif BOOLEAN DEFAULT TRUE
);
```

**Données actuelles :**

| Code | Nom | Description |
|------|-----|-------------|
| INST | Instrumentiste | Prépare et passe les instruments |
| CIRC | Circulante | Coordonne la salle, documentation |
| AIDE_OP | Aide opératoire | Assiste le chirurgien |
| AIDE_GASTRO | Aide bloc gastro | Spécifique endoscopies |

---

## 5.2 Table `secretaires_besoins_operations`

Préférences des secrétaires pour les besoins d'opération. **Sans préférence = non éligible.**

```sql
CREATE TABLE secretaires_besoins_operations (
    secretaire_id UUID REFERENCES secretaires(id),
    besoin_operation_id UUID REFERENCES besoins_operations(id),
    preference INTEGER NOT NULL,  -- 1 = P1, 2 = P2, 3 = P3
    PRIMARY KEY (secretaire_id, besoin_operation_id)
);
```

| Priorité | Score Soft | Éligibilité |
|----------|------------|-------------|
| P1 | +100 | ✓ Éligible |
| P2 | +80 | ✓ Éligible |
| P3 | +60 | ✓ Éligible |
| Aucune | - | ✗ **Non éligible (contrainte HARD)** |

---

## 5.3 Table `types_intervention`

Types d'intervention chirurgicale.

```sql
CREATE TABLE types_intervention (
    id UUID PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    code VARCHAR(20),
    specialite VARCHAR(50),
    salle_preferentielle VARCHAR(20),
    salle_exclusive VARCHAR(20),
    actif BOOLEAN DEFAULT TRUE
);
```

---

## 5.4 Table `types_intervention_besoins_personnel`

Définit les besoins en personnel pour chaque type d'intervention.

```sql
CREATE TABLE types_intervention_besoins_personnel (
    id UUID PRIMARY KEY,
    type_intervention_id UUID REFERENCES types_intervention(id),
    besoin_operation_id UUID REFERENCES besoins_operations(id),
    nombre_requis INTEGER NOT NULL,
    actif BOOLEAN DEFAULT TRUE
);
```

**Exemple :**

| Type d'intervention | Besoin | Nombre |
|---------------------|--------|--------|
| Chirurgie Cataracte | Instrumentiste | 1 |
| Chirurgie Cataracte | Circulante | 1 |
| Endoscopie digestive | Aide bloc gastro | 1 |
| Endoscopie digestive | Circulante | 1 |
| Chirurgie générale | Instrumentiste | 1 |
| Chirurgie générale | Circulante | 1 |
| Chirurgie générale | Aide opératoire | 1 |

---

## 5.5 Table `planning_genere_bloc_operatoire`

Planning des interventions chirurgicales.

```sql
CREATE TABLE planning_genere_bloc_operatoire (
    id UUID PRIMARY KEY,
    date DATE NOT NULL,
    periode VARCHAR(20) NOT NULL,  -- 'matin', 'apres_midi'
    type_intervention_id UUID REFERENCES types_intervention(id),
    medecin_id UUID REFERENCES medecins(id),
    salle_assignee VARCHAR(20),  -- 'salle_rouge', 'salle_verte', 'salle_jaune', 'salle_gastro'
    validated BOOLEAN DEFAULT FALSE
);
```

**Salles disponibles :**

| Salle | Types d'intervention |
|-------|---------------------|
| salle_rouge | Chirurgie générale, Cataracte |
| salle_verte | Chirurgie générale, Angiologie |
| salle_jaune | Chirurgie générale |
| salle_gastro | Endoscopies digestives |

**Calcul de la demande bloc :**

```
Pour chaque opération planifiée :
1. Récupérer type_intervention_id
2. Chercher dans types_intervention_besoins_personnel
3. Pour chaque besoin_operation requis :
   → Créer une demande de N personnes avec ce besoin_operation
```

---

# 6. RÉSULTAT (Output)

## 6.1 Table `capacite_effective`

Stocke les assignations générées par le solveur.

```sql
CREATE TABLE capacite_effective (
    id UUID PRIMARY KEY,
    date DATE NOT NULL,
    demi_journee VARCHAR(20) NOT NULL,
    secretaire_id UUID REFERENCES secretaires(id),
    site_id UUID REFERENCES sites(id),

    -- Pour consultations
    skill_id UUID REFERENCES skills(id),

    -- Pour bloc opératoire
    besoin_operation_id UUID REFERENCES besoins_operations(id),
    planning_genere_bloc_operatoire_id UUID REFERENCES planning_genere_bloc_operatoire(id),

    -- Rôles de fermeture
    is_1r BOOLEAN DEFAULT FALSE,
    is_2f BOOLEAN DEFAULT FALSE,
    is_3f BOOLEAN DEFAULT FALSE
);
```

---

# 7. RELATIONS ET INTÉGRITÉ

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    sites     │────<│ emplacements │────<│  specialites │
└──────────────┘     └──────────────┘     └──────────────┘
                                                  │
                                                  ▼
                     ┌──────────────┐     ┌──────────────┐
                     │   medecins   │────>│    skills    │
                     └──────────────┘     └──────────────┘
                            │                     ▲
                            ▼                     │
                     ┌──────────────────┐         │
                     │ planning_medecins│         │
                     └──────────────────┘         │
                                                  │
┌──────────────┐     ┌──────────────────────────┐ │
│ secretaires  │────<│ secretaires_pref_skill   │─┘
└──────────────┘     └──────────────────────────┘
       │
       ├────<│ secretaires_pref_site    │
       │
       ├────<│ secretaires_pref_medecin │
       │
       ├────<│ absences                 │
       │
       ├────<│ shifts_fixes             │
       │
       └────<│ secretaires_besoins_op   │─────>│ besoins_operations │
                                                         ▲
                                                         │
       ┌───────────────────────────────────────┐         │
       │ types_intervention_besoins_personnel  │─────────┘
       └───────────────────────────────────────┘
                            │
                            ▼
       ┌───────────────────────────────────────┐
       │        types_intervention             │
       └───────────────────────────────────────┘
                            ▲
                            │
       ┌───────────────────────────────────────┐
       │   planning_genere_bloc_operatoire     │
       └───────────────────────────────────────┘
```

---

# 8. RÉSUMÉ DES TABLES

| Table | Description | Fréquence MAJ |
|-------|-------------|---------------|
| `sites` | Lieux géographiques | Rare |
| `emplacements` | Services/départements | Rare |
| `specialites` | Types de staffing | Rare |
| `skills` | Compétences consultation | Rare |
| `besoins_operations` | Rôles bloc opératoire | Rare |
| `types_intervention` | Types de chirurgie | Rare |
| `types_intervention_besoins_personnel` | Besoins par intervention | Rare |
| `secretaires` | Secrétaires médicales | Hebdomadaire |
| `secretaires_preferences_*` | Préférences | Occasionnel |
| `secretaires_besoins_operations` | Préférences bloc | Occasionnel |
| `absences` | Indisponibilités | Hebdomadaire |
| `shifts_fixes` | Horaires prédéfinis | Hebdomadaire |
| `medecins` | Médecins | Rare |
| `planning_medecins` | Planning consultations | Hebdomadaire |
| `planning_genere_bloc_operatoire` | Planning opérations | Hebdomadaire |
| `capacite_effective` | **OUTPUT** - Assignations | Généré |

---

*Document de spécification du schéma de base de données*
*Voir aussi : CONSTRAINTS.md, OUTPUT_SCHEMA.md*
