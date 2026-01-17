# Contraintes et Règles de Scoring OptaPlanner

Ce document décrit **toutes les contraintes** et règles de scoring du solveur.

---

# 1. STRUCTURE DU SCORE

## 1.1 HardMediumSoftScore

OptaPlanner utilise une **comparaison lexicographique** sur 3 niveaux :

```
Score = (Hard, Medium, Soft)

Comparaison :
  (0, 0, -1000) > (-1, 0, +999999)   // Hard prime sur tout
  (0, 0, -1000) > (0, -1, +999999)   // Medium prime sur Soft
```

| Niveau | Rôle | Objectif |
|--------|------|----------|
| **HARD** | Faisabilité | = 0 obligatoire |
| **MEDIUM** | Couverture | = 0 souhaitable |
| **SOFT** | Qualité | Maximiser |

## 1.2 Principe du Degré de Violation

Toujours spécifier le **degré** de violation pour guider le solveur :

```java
// MAUVAIS : pas de gradient
if (violation) → -100h

// BON : gradient pour le solveur
for each violation → -100h
```

---

# 2. CONTRAINTES HARD (Faisabilité)

**Score hard < 0 = solution INVALIDE**

## 2.1 Liste Complète

| ID | Contrainte | Règle | Pénalité |
|----|------------|-------|----------|
| H1 | **Conflit temporel** | 1 ressource = 1 shift max par période | -100h par conflit |
| H2 | **Éligibilité Skill** | Ressource doit posséder le skill requis | -100h par violation |
| H2b | **Éligibilité Site** | Ressource doit avoir le site dans ses préférences | -100h par violation |
| H3 | **Exclusion Bloc-Site distant** | Bloc + Site distant même jour = interdit | -100h par violation |
| H4 | **Florence Bron 2F mardi** | Interdiction absolue | -100h |
| H5 | **Lucie Pratillo 2F/3F** | Interdiction absolue | -100h |
| H6 | **Jours exacts (flexible)** | Ressource flexible = exactement `nombre_jours_semaine` | -100h par jour de différence |
| H7 | **Absence** | Ressource ne peut pas être assignée sur créneau d'absence | -100h par violation |
| H8 | **Continuité fermeture** | 1R/2F = même personne matin ET après-midi | -100h par violation |

## 2.2 Détails des Contraintes

### H1 - Conflit Temporel

Une ressource ne peut être assignée qu'à **1 seul shift** par période.

```
Violation : Marie assignée à Ophtalmo ET Dermato le lundi matin
Pénalité : -100h
```

### H2 - Éligibilité Skill

La ressource doit avoir le skill dans ses préférences pour être assignée.

```
Violation : Pierre assigné à Accueil Dermato mais n'a pas ce skill
Pénalité : -100h
```

**Pour le Bloc Opératoire** : La ressource doit avoir une préférence pour le `besoin_operation` requis.

```
Violation : Marie assignée comme Instrumentiste mais n'a pas ce besoin_operation
Pénalité : -100h
```

### H2b - Éligibilité Site

La ressource doit avoir le site dans ses préférences (P1, P2, P3 ou P4) pour être assignée.

```
Violation : Pierre assigné à Porrentruy mais n'a pas ce site dans ses préférences
Pénalité : -100h
```

**Note** : Une ressource sans préférence de site pour un emplacement donné ne peut PAS y être assignée.

### H3 - Exclusion Bloc-Site Distant

Combinaisons **interdites** le même jour :

| Matin | Après-midi | Interdit |
|-------|------------|----------|
| Bloc opératoire | Porrentruy | ✓ |
| Bloc opératoire | Vieille Ville | ✓ |
| Porrentruy | Bloc opératoire | ✓ |
| Vieille Ville | Bloc opératoire | ✓ |

### H4 & H5 - Règles Individuelles

| Ressource | UUID | Interdiction |
|-----------|------|--------------|
| Florence Bron | `1e5339aa-5e82-4295-b918-e15a580b3396` | 2F le mardi |
| Lucie Pratillo | `5d3af9e3-674b-48d6-b54f-bd84c9eee670` | 2F et 3F toujours |

### H6 - Jours Exacts (Ressources Flexibles)

Pour `horaire_flexible = true` :

```
nombre_jours_assignés DOIT == nombre_jours_semaine

Exemple :
  Marie : nombre_jours_semaine = 4
  Assignée 3 jours → -100h (1 jour de différence)
  Assignée 5 jours → -100h (1 jour de différence)
```

### H7 - Absence

Une ressource avec absence ne peut **jamais** être assignée sur ce créneau.

```
Marie : absence mercredi (jour complet)
Violation : Marie assignée mercredi matin
Pénalité : -100h
```

### H8 - Continuité Fermeture

Sur un site avec fermeture, si médecins présents **matin ET après-midi** :

```
Les rôles 1R et 2F doivent être la MÊME personne toute la journée

Exemple (VALIDE) :
  Ophtalmo lundi matin : Marie (1R), Pierre (2F)
  Ophtalmo lundi PM : Marie (1R), Pierre (2F) ✓

Exemple (INVALIDE) :
  Ophtalmo lundi matin : Marie (1R), Pierre (2F)
  Ophtalmo lundi PM : Anna (1R), Luc (2F) ✗ → -200h (2 violations)
```

---

# 3. CONTRAINTES MEDIUM (Couverture)

**Score medium < 0 = besoins non satisfaits**

## 3.1 Liste Complète

| ID | Contrainte | Règle | Pénalité |
|----|------------|-------|----------|
| M1 | **Skill manquant** | Chaque skill requis doit être couvert | **-1000m** par skill |
| M2 | **Fermeture non couverte** | Sites avec fermeture sans 1R ou 2F | **-1000m** par rôle |

## 3.2 Justification de -1000m

La pénalité Medium doit être **supérieure** au pire cumul Soft :

```
Pire cumul Soft ≈ 810 :
  - P234 jour 5+ : -200
  - Fermeture palier 4 : -500
  - Cumul : -50
  - Changement site : -20
  - Perte préférence : -40

Avec -1000m :
  Le solveur préfère TOUJOURS remplir un shift (même avec -810 soft)
  plutôt que le laisser vide (-1000m) et mettre la ressource en Admin (+15 soft)
```

## 3.3 Exemples

### M1 - Skill Manquant

```
Dermato lundi matin avec 2 médecins :
  Besoin : 2 Aide Dermato + 2 Accueil Dermato = 4 personnes
  Assigné : 2 Aide + 1 Accueil = 3 personnes

  Violation : 1 Accueil Dermato manquant
  Pénalité : -1000m
```

### M2 - Fermeture Non Couverte

```
Ophtalmologie lundi (médecins matin + PM) :
  Besoin : 1R + 2F
  Assigné : 1R uniquement

  Violation : 2F manquant
  Pénalité : -1000m
```

---

# 4. CONTRAINTES SOFT (Qualité)

**Score soft = qualité de la solution**

## 4.1 Préférences (3 Dimensions)

On prend le **MAX** des 3 scores pour chaque assignation.

| Dimension | P1 | P2 | P3 | P4 |
|-----------|-----|-----|-----|-----|
| **Skill** | +100 | +80 | +60 | - |
| **Médecin** | +70 | +50 | - | - |
| **Site** | +40 | +35 | +30 | +25 |

```
Score_Assignation = max(Score_Skill, Score_Médecin, Score_Site)
```

**Exemple** :
```
Marie assignée à Ophtalmo La Vallée avec Dr. Martin :
  - Skill Accueil Ophtalmo : P1 → +100
  - Site La Vallée : P1 → +40
  - Médecin Dr. Martin : P2 → +50

Score = max(100, 40, 50) = +100 soft
```

## 4.2 Continuité de Site

| Situation | Score |
|-----------|-------|
| Même site matin + après-midi | **+20** |
| Changement de site | **-20** |
| Admin impliqué (un côté ou l'autre) | **0** |

```
Marie : Ophtalmo matin + Ophtalmo PM → +20 soft
Pierre : Dermato matin + Porrentruy PM → -20 soft
Anna : Ophtalmo matin + Admin PM → 0 soft
```

## 4.3 Équité de Charge (Fairness)

**Formule quadratique** pour inciter à l'équilibrage :

```
score_fairness -= Σ (charge_r)²
```

**Pourquoi quadratique ?**
```
Transférer 1 shift de Marie (5 shifts) vers Pierre (2 shifts) :
  Avant : 5² + 2² = 29
  Après : 4² + 3² = 25
  Amélioration : +4 soft ✓
```

## 4.4 Équilibrage Sites Distants (P234)

Pour ressources dont le site est en P2/P3/P4 :

| Jour # au site distant | Pénalité |
|------------------------|----------|
| 1er | 0 |
| 2ème | -20 |
| 3ème | -50 |
| 4ème | -100 |
| 5ème+ | -200 |

```
Marie (Porrentruy = P3) :
  Jour 1 : 0
  Jour 2 : -20
  Jour 3 : -50
  Total si 3 jours : -70 soft
```

## 4.5 Équilibrage Rôles Fermeture

**Score de charge** = jours_1R × 10 + jours_2F × 12

| Score | Pénalité |
|-------|----------|
| ≤ 22 | 0 |
| 23-29 | -30 |
| 30-31 | -80 |
| 32-35 | -150 |
| > 35 | -500 |

*Seul le palier le plus élevé s'applique.*

```
Marie cette semaine : 2 jours 1R + 1 jour 2F
Score = 2×10 + 1×12 = 32 → -150 soft
```

## 4.6 Bonus Administratif

### Ressources avec `prefered_admin = true`

| Condition | Bonus |
|-----------|-------|
| Shift admin jusqu'à `nombre_demi_journees_admin` | +15 par shift |
| Shift admin au-delà | +5 par shift |

### Ressources avec `prefered_admin = false`

Score décroissant : +10, +9, +8, +7...

```
Pierre (prefered_admin = false), 3 shifts admin :
  Shift 1 : +10
  Shift 2 : +9
  Shift 3 : +8
  Total : +27 soft
```

## 4.7 Cumul P234 + Fermeture

Si une ressource a **simultanément** :
- Pénalité P234 active (≥2 jours site distant)
- Pénalité fermeture active (score > 22)

**Pénalité additionnelle** : -50 soft

## 4.8 Préférence Admin pour Rôle Fermeture (Demi-journée)

Si le site avec fermeture n'a des médecins que **matin OU après-midi** :

| Situation | Bonus |
|-----------|-------|
| Ressource 1R/2F le matin → Admin l'après-midi | +30 |
| Ressource 1R/2F l'après-midi → Admin le matin | +30 |

---

# 5. RÈGLES SPÉCIALES

## 5.1 Paul Jacquier et le rôle 3F

| Ressource | UUID |
|-----------|------|
| Paul Jacquier | `121dc7d9-99dc-46bd-9b6c-d240ac6dc6c8` |

**Règle** : Si Paul Jacquier est présent **jeudi ET vendredi** sur un site avec fermeture, un rôle **3F** est requis.

## 5.2 Bonus Médecin-Secrétaire

| Médecin | Secrétaires | Bonus |
|---------|-------------|-------|
| Dr. fda323f4 (`fda323f4-3efd-4c78-8b63-7d660fcd7eea`) | Sara Bortolon, Mirlinda Hasani | +50 soft |

| Secrétaire | UUID |
|------------|------|
| Sara Bortolon | `68e74e31-12a7-4fd3-836d-41e8abf57792` |
| Mirlinda Hasani | `324639fa-2e3d-4903-a143-323a17b0d988` |

## 5.3 Obstétricienne seule sur site Ophtalmologie

Si un site Ophtalmo n'a **aucun médecin standard** mais a **≥1 obstétricienne** :

```
Demande = 1 Accueil Ophtalmo (au lieu de 0)
```

---

# 6. RÉCAPITULATIF DES SCORES

## 6.1 Contraintes Hard (-100h chacune)

- H1: Conflit temporel
- H2: Éligibilité skill
- H3: Exclusion Bloc-Site distant
- H4: Florence Bron 2F mardi
- H5: Lucie Pratillo 2F/3F
- H6: Jours exacts (flexible)
- H7: Absence
- H8: Continuité fermeture

## 6.2 Contraintes Medium

| Contrainte | Pénalité |
|------------|----------|
| Skill manquant | -1000m |
| Rôle fermeture manquant | -1000m |

## 6.3 Contraintes Soft - Bonus

| Élément | Score |
|---------|-------|
| Skill P1 | +100 |
| Skill P2 | +80 |
| Skill P3 | +60 |
| Médecin P1 | +70 |
| Médecin P2 | +50 |
| Site P1 | +40 |
| Site P2 | +35 |
| Site P3 | +30 |
| Site P4 | +25 |
| Continuité site | +20 |
| Admin (préféré) | +15 |
| Admin (non préféré) | +10 décroissant |
| Admin si rôle fermeture demi-journée | +30 |
| Bonus Dr. fda323f4 | +50 |

## 6.4 Contraintes Soft - Pénalités

| Élément | Score |
|---------|-------|
| Changement de site | -20 |
| P234 jour 2 | -20 |
| P234 jour 3 | -50 |
| P234 jour 4 | -100 |
| P234 jour 5+ | -200 |
| Fermeture palier 1 (23-29) | -30 |
| Fermeture palier 2 (30-31) | -80 |
| Fermeture palier 3 (32-35) | -150 |
| Fermeture palier 4 (>35) | -500 |
| Cumul P234 + fermeture | -50 |
| Fairness | -Σ(charge²) |

---

# 7. FORMULE GLOBALE

```
Score = HardMediumSoftScore(

   hard = Σ violations_H1_à_H8 × (-100),

   medium = Σ skills_manquants × (-1000)
          + Σ rôles_fermeture_manquants × (-1000),

   soft = Σ max(score_skill, score_médecin, score_site)  // Préférences
        + Σ bonus_continuité_site                         // +20 si même site
        + Σ bonus_admin                                   // +15 ou décroissant
        + Σ bonus_fermeture_admin                         // +30 si demi-journée
        + Σ bonus_médecin_spécial                         // +50 Dr. fda323f4
        - Σ pénalités_changement_site                     // -20
        - Σ pénalités_P234                                // -20 à -200
        - Σ pénalités_fermeture                           // -30 à -500
        - Σ pénalités_cumul                               // -50
        - Σ (charge_r)²                                   // Fairness quadratique
)
```

---

*Document de spécification des contraintes OptaPlanner*
*Voir aussi : INPUT_SCHEMA.md, OUTPUT_SCHEMA.md*
