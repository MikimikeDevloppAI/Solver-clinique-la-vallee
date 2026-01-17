# Modèle de Données - Planification des Secrétaires Médicales

Ce document décrit la structure des données pour le solveur OptaPlanner.

---

## Vue d'Ensemble

```
┌─────────────────────────────────────────────────────────────────┐
│                      MODÈLE DE DONNÉES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STRUCTURE GÉOGRAPHIQUE                                         │
│  =====================                                          │
│  sites                                                          │
│    └── emplacements                                             │
│          └── specialites                                        │
│                └── skills                                       │
│                                                                  │
│  PERSONNES                                                      │
│  =========                                                      │
│  medecins (besoin_secretaires, specialite)                      │
│  secretaires (préférences, disponibilités)                      │
│                                                                  │
│  PLANNING UNIFIÉ                                                │
│  ===============                                                │
│  planning                                                       │
│    ├── type = 'medecin' | 'secretaire'                          │
│    ├── personne_id                                              │
│    ├── site_id, date, demi_journee                              │
│    └── skill_id, operation_id (selon contexte)                  │
│                                                                  │
│  BLOC OPÉRATOIRE                                                │
│  ===============                                                │
│  operations                                                     │
│  types_intervention → besoins_personnel                         │
│  besoins_operations                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

# 1. STRUCTURE GÉOGRAPHIQUE

## 1.1 Sites

Les **sites** sont les lieux géographiques où travaillent les personnes.

| ID | Nom | Type distance |
|----|-----|---------------|
| `site-la-vallee` | Clinique La Vallée | reference (site principal) |
| `site-porrentruy` | Porrentruy | distant |
| `site-vieille-ville` | Vieille Ville Delémont | proche |

**Champs** : id, nom, adresse, distance_type

---

## 1.2 Emplacements

Les **emplacements** sont les services/départements dans un site.

| UUID | Nom | Site | Type | Fermeture |
|------|-----|------|------|-----------|
| `7c8abe96-0a6b-44eb-857f-ad69036ebc88` | Ophtalmologie | La Vallée | consultation | ✓ |
| `d82c55ee-2964-49d4-a578-417b55b557ec` | Dermatologie | La Vallée | consultation | ✗ |
| `c3bd13...` | Angiologie | La Vallée | consultation | ✗ |
| `0e1b31...` | Rhumatologie | La Vallée | consultation | ✗ |
| `86f1047f-c4ff-441f-a064-42ee2f8ef37a` | Bloc opératoire | La Vallée | bloc | ✗ |
| `7723c334-d06c-413d-96f0-be281d76520d` | Gastroentérologie | Vieille Ville | consultation | ✗ |
| `043899a1-a232-4c4b-9d7d-0eb44dad00ad` | Porrentruy | Porrentruy | consultation | ✓ |
| `00000000-0000-0000-0000-000000000001` | Administratif | - | admin | ✗ |

**Champs** : id, nom, site_id, type, specialite_id, fermeture

### Types d'emplacements

| Type | Description | Emplacements | Caractéristique |
|------|-------------|--------------|-----------------|
| **A** | Ratio fixe Aide/Accueil | Gastro, Dermato | Minimum par skill selon nb médecins |
| **B** | Formule standard | Ophtalmo, Angio, Rhuma, ORL | Skill unique, ceil(Σ besoin) |
| **C** | Skills par intervention | Bloc opératoire | Besoins définis par type d'intervention |
| **D** | Sans skill requis | Admin | Capacité illimitée, ouvert à tous |

---

## 1.3 Spécialités et Skills

Chaque spécialité a ses **skills propres**. Une secrétaire doit avoir le skill dans ses préférences pour être éligible.

### Skills de consultation

| Spécialité | Skill Aide de salle | Skill Accueil |
|------------|---------------------|---------------|
| Gastroentérologie | Aide de salle Gastro | Accueil Gastro |
| Dermatologie | Aide de salle Dermato | Accueil Dermato |
| Ophtalmologie | - | Accueil Ophtalmo |
| Angiologie | - | Accueil Angio |
| Rhumatologie | - | Accueil Rhuma |
| ORL | - | Accueil ORL |
| Gynécologie | - | Accueil Gynéco |

### Skills de fermeture

| Skill | Description |
|-------|-------------|
| 1R | Première Responsabilité |
| 2F | Deuxième Fermeture |
| 3F | Troisième Fermeture (cas spécial Paul Jacquier) |

### Skills de bloc opératoire

Définis par la table `types_intervention_besoins_personnel`. Voir section Bloc Opératoire.

---

# 2. PERSONNES

## 2.1 Médecins

**Champs** : id, nom, prenom, specialite_id, besoin_secretaires, is_obstetricienne

| Champ | Description |
|-------|-------------|
| `besoin_secretaires` | Coefficient de besoin (ex: 1.0, 1.2, 1.5, 2.0) |
| `is_obstetricienne` | Exception : besoin = 0, ne compte pas dans le calcul |

---

## 2.2 Secrétaires

**Champs** : id, nom, prenom, active, horaire_flexible, nombre_jours_semaine, prefered_admin, nombre_demi_journees_admin

### Deux types d'horaires

| Type | `horaire_flexible` | Données | Solveur décide |
|------|-------------------|---------|----------------|
| **Flexible** | `true` | `nombre_jours_semaine` + absences | Quels jours + où |
| **Fixe** | `false` | Shifts prédéfinis dans planning | Seulement où |

### Préférences des secrétaires

Les préférences déterminent l'**éligibilité** ET le **score**.

| Table | Champs | Éligibilité |
|-------|--------|-------------|
| `preferences_site` | secretaire_id, site_id, priorite (P1-P4) | Sans préférence = **non éligible** |
| `preferences_skill` | secretaire_id, skill_id, priorite (P1-P3) | Sans préférence = **non éligible** |
| `preferences_medecin` | secretaire_id, medecin_id, priorite (P1-P2) | Optionnel (bonus) |
| `preferences_besoin_operation` | secretaire_id, besoin_operation_id, priorite | Pour le bloc |

### Secrétaires avec règles spéciales

| UUID | Nom | Règle |
|------|-----|-------|
| `1e5339aa-5e82-4295-b918-e15a580b3396` | Florence Bron | Pas 2F le mardi |
| `5d3af9e3-674b-48d6-b54f-bd84c9eee670` | Lucie Pratillo | Jamais 2F ni 3F |
| `121dc7d9-99dc-46bd-9b6c-d240ac6dc6c8` | Paul Jacquier | Déclenche 3F si présent jeudi+vendredi |
| `68e74e31-12a7-4fd3-836d-41e8abf57792` | Sara Bortolon | Bonus avec Dr. fda323f4 |
| `324639fa-2e3d-4903-a143-323a17b0d988` | Mirlinda Hasani | Bonus avec Dr. fda323f4 |

---

# 3. PLANNING UNIFIÉ

Table unique contenant le planning des médecins ET des secrétaires.

**Champs** :
- id, date, demi_journee (`matin` | `apres_midi`)
- site_id
- **type** : `medecin` | `secretaire`
- **personne_id** : UUID du médecin ou de la secrétaire
- skill_id (pour secrétaires en consultation)
- operation_id, besoin_operation_id (pour bloc)
- is_1r, is_2f, is_3f (rôles fermeture)

### Utilisation

| type | Rôle |
|------|------|
| `medecin` | **INPUT** - Planning des médecins (source des besoins) |
| `secretaire` | **OUTPUT** - Assignations générées par le solveur |

---

# 4. BLOC OPÉRATOIRE

## 4.1 Opérations

Planning des interventions chirurgicales.

**Champs** : id, date, periode, type_intervention_id, medecin_id, salle_assignee

### Salles disponibles

| Salle | Types d'intervention |
|-------|---------------------|
| salle_rouge | Chirurgie générale, Cataracte |
| salle_verte | Chirurgie générale, Angiologie |
| salle_jaune | Chirurgie générale |
| salle_gastro | Endoscopies digestives |

## 4.2 Types d'intervention

Définition des interventions chirurgicales.

**Champs** : id, nom, code, specialite

## 4.3 Besoins en personnel par intervention

Chaque type d'intervention définit les besoins en personnel.

**Champs** : type_intervention_id, besoin_operation_id, nombre_requis

**Exemple** :

| Type d'intervention | Besoin | Nombre |
|---------------------|--------|--------|
| Chirurgie Cataracte | Instrumentiste | 1 |
| Chirurgie Cataracte | Circulante | 1 |
| Endoscopie digestive | Aide bloc gastro | 1 |
| Endoscopie digestive | Circulante | 1 |
| Chirurgie générale | Instrumentiste | 1 |
| Chirurgie générale | Circulante | 1 |
| Chirurgie générale | Aide opératoire | 1 |

## 4.4 Besoins d'opération

Rôles disponibles au bloc (équivalent des skills).

| Code | Nom | Description |
|------|-----|-------------|
| INST | Instrumentiste | Prépare et passe les instruments |
| CIRC | Circulante | Coordonne la salle, documentation |
| AIDE_OP | Aide opératoire | Assiste le chirurgien |
| AIDE_GASTRO | Aide bloc gastro | Spécifique endoscopies |

---

# 5. ABSENCES ET DISPONIBILITÉS

## 5.1 Absences (horaire flexible)

Pour les secrétaires avec `horaire_flexible = true`.

**Champs** : secretaire_id, date, periode (`matin` | `apres_midi` | null = jour complet)

**Exemple** :
```
Marie (flexible, 4 jours/semaine) :
  - Absence mercredi (jour complet)
  - Absence vendredi matin

→ Jours disponibles : lundi, mardi, jeudi, vendredi PM, samedi
→ Solveur choisit 4 jours parmi ceux-ci
```

## 5.2 Shifts fixes (horaire fixe)

Pour les secrétaires avec `horaire_flexible = false`, les shifts sont prédéfinis dans la table planning.

**Exemple** :
```
Pierre (fixe) - entrées dans planning (type='secretaire', site=null) :
  - Lundi matin
  - Lundi après-midi
  - Mardi matin
  - Jeudi après-midi

→ Solveur DOIT assigner Pierre sur ces 4 créneaux
→ Solveur choisit le site/emplacement pour chaque créneau
```

---

# 6. IDs DE RÉFÉRENCE

## Sites

| ID | Nom |
|----|-----|
| `00000000-0000-0000-0000-000000000001` | Administratif |
| `7723c334-d06c-413d-96f0-be281d76520d` | Vieille Ville (Gastro) |
| `7c8abe96-0a6b-44eb-857f-ad69036ebc88` | Ophtalmologie |
| `d82c55ee-2964-49d4-a578-417b55b557ec` | Dermatologie |
| `043899a1-a232-4c4b-9d7d-0eb44dad00ad` | Porrentruy |
| `86f1047f-c4ff-441f-a064-42ee2f8ef37a` | Bloc opératoire |

---

*Voir aussi : 2_NEEDS_CALCULATION.md, 3_CONSTRAINTS.md, 4_OUTPUT.md*
