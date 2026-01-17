# Calcul des Besoins en Secrétaires

Ce document explique comment calculer les besoins en secrétaires à partir du planning des médecins.

---

## Vue d'Ensemble

```
ENTRÉE                              SORTIE
========                            ======
planning (type='medecin')    →      Besoins par (site, date, demi_journee)
operations (bloc)            →        ├── nombre de secrétaires
                                      ├── skills requis
                                      └── rôles fermeture
```

---

# 1. FLUX DE CALCUL

## Étape 1 : Lire le planning des médecins

```
SELECT * FROM planning
WHERE type = 'medecin'
  AND date BETWEEN date_debut AND date_fin
```

## Étape 2 : Grouper par (site, date, demi_journee)

Pour chaque groupe, lister les médecins présents avec leur `besoin_secretaires`.

## Étape 3 : Calculer le besoin brut

```
besoin_brut = Σ medecins.besoin_secretaires
```

**Exception** : Les obstétriciennes (`is_obstetricienne = true`) ont un `besoin_secretaires = 0` et ne comptent pas.

## Étape 4 : Appliquer les règles par type d'emplacement

Voir section 2.

## Étape 5 : Ajouter les besoins de fermeture

Voir section 3.

---

# 2. RÈGLES PAR TYPE D'EMPLACEMENT

## Type A : Ratio Fixe (Gastro, Dermato)

### Gastro (Vieille Ville)

| Condition | Aide de salle Gastro | Accueil Gastro | Total |
|-----------|---------------------|----------------|-------|
| ≥1 médecin | 2 | 1 | **3 minimum** |

### Dermato (La Vallée)

| Condition | Aide de salle Dermato | Accueil Dermato | Total |
|-----------|----------------------|-----------------|-------|
| 1 médecin | 2 | 1 | **3 minimum** |
| ≥2 médecins | 2 | 2 | **4 minimum** |

**Formule finale** : `max(ceil(besoin_brut), minimum_type_A)`

---

## Type B : Formule Standard (Ophtalmo, Angio, Rhuma, ORL)

| Condition | Demande |
|-----------|---------|
| 0 médecin | 0 |
| ≥1 médecin | `max(1, ceil(besoin_brut))` |

**Skill** : Accueil de la spécialité (ex: Accueil Ophtalmo)

---

## Type C : Bloc Opératoire

Le calcul est différent : basé sur les **opérations** planifiées.

### Flux

```
1. Lire operations du jour
2. Pour chaque operation :
   - Récupérer type_intervention_id
   - Chercher dans types_intervention_besoins_personnel
   - Créer un besoin par besoin_operation requis
3. Agréger les besoins par (date, demi_journee, besoin_operation)
```

### Exemple

```
Bloc - Lundi matin
==================

Opérations planifiées :
  - Salle Rouge : Chirurgie Cataracte
  - Salle Gastro : Endoscopie digestive

Besoins par intervention :
  Cataracte     → 1 Instrumentiste + 1 Circulante
  Endoscopie    → 1 Aide bloc gastro + 1 Circulante

Besoins totaux :
  - 1 × Instrumentiste
  - 1 × Aide bloc gastro
  - 2 × Circulante (ou 1 si même personne couvre les 2 salles)
```

---

## Type D : Admin

- Pas de calcul de besoin
- Capacité illimitée
- Aucun skill requis
- Toute secrétaire peut être assignée

---

# 3. BESOINS DE FERMETURE

Pour les emplacements avec `fermeture = true` (Ophtalmo, Porrentruy).

## Règle

Si des médecins sont présents sur un emplacement avec fermeture :
- Rôle **1R** (Première Responsabilité) requis
- Rôle **2F** (Deuxième Fermeture) requis

## Journée complète vs demi-journée

| Situation | Règle |
|-----------|-------|
| Médecins matin ET après-midi | 1R et 2F doivent être la **MÊME personne** toute la journée |
| Médecins matin uniquement | 1R et 2F le matin, bonus si Admin l'après-midi |
| Médecins après-midi uniquement | 1R et 2F l'après-midi, bonus si Admin le matin |

## Cas spécial : Paul Jacquier

Si Paul Jacquier (`121dc7d9-99dc-46bd-9b6c-d240ac6dc6c8`) est présent **jeudi ET vendredi** sur un site avec fermeture :
- Rôle **3F** (Troisième Fermeture) requis en plus

---

# 4. EXCEPTIONS

## Samedi

```
besoin = nombre_médecins (pas de ceiling, pas de minimum)
```

## Obstétriciennes seules sur Ophtalmo

Si un site Ophtalmo n'a **aucun médecin standard** mais a **≥1 obstétricienne** :

```
besoin = 1 Accueil Ophtalmo
```

---

# 5. EXEMPLES COMPLETS

## Exemple 1 : Dermato avec 2 médecins

```
Dermato - Lundi 5 janvier - Matin
=================================

INPUT (planning WHERE type='medecin'):
  - Dr. Martin (besoin_secretaires = 1.5)
  - Dr. Dupont (besoin_secretaires = 1.8)

CALCUL:
  1. besoin_brut = 1.5 + 1.8 = 3.3
  2. ceil(3.3) = 4
  3. Dermato avec 2+ médecins → minimum 4
  4. max(4, 4) = 4 secrétaires

BESOINS GÉNÉRÉS:
  - 2 × Aide de salle Dermato
  - 2 × Accueil Dermato
```

## Exemple 2 : Ophtalmo avec fermeture

```
Ophtalmo - Mardi 6 janvier
==========================

Matin:
  - Dr. Bernard (besoin_secretaires = 1.2)
  - Dr. Thomas (besoin_secretaires = 1.2)

Après-midi:
  - Dr. Bernard (besoin_secretaires = 1.2)

CALCUL:
  Matin: ceil(2.4) = 3 Accueil Ophtalmo
  Après-midi: ceil(1.2) = 2 Accueil Ophtalmo

FERMETURE (médecins matin ET PM):
  - 1R requis (même personne toute la journée)
  - 2F requis (même personne toute la journée)

BESOINS GÉNÉRÉS:
  Matin: 3 × Accueil Ophtalmo + 1R + 2F
  Après-midi: 2 × Accueil Ophtalmo + 1R + 2F (mêmes personnes)
```

## Exemple 3 : Bloc opératoire

```
Bloc - Mercredi 7 janvier - Matin
=================================

OPÉRATIONS:
  - 08:00 Salle Rouge : Chirurgie générale (Dr. Lefevre)
  - 08:30 Salle Gastro : Endoscopie (Dr. Dubois)
  - 10:00 Salle Verte : Cataracte (Dr. Martin)

BESOINS PAR OPÉRATION:
  Chirurgie générale → 1 Instrumentiste + 1 Circulante + 1 Aide opératoire
  Endoscopie → 1 Aide bloc gastro + 1 Circulante
  Cataracte → 1 Instrumentiste + 1 Circulante

AGRÉGATION:
  - 2 × Instrumentiste (Rouge + Verte en parallèle)
  - 1 × Aide bloc gastro
  - 1 × Aide opératoire
  - 3 × Circulante (ou moins si une personne peut couvrir plusieurs salles)
```

## Exemple 4 : Samedi

```
Ophtalmo - Samedi 10 janvier - Matin
====================================

INPUT:
  - Dr. Martin (besoin_secretaires = 1.2)
  - Dr. Dupont (besoin_secretaires = 1.5)

CALCUL SAMEDI:
  besoin = 2 (nombre de médecins, pas ceil(2.7))

BESOINS GÉNÉRÉS:
  - 2 × Accueil Ophtalmo
```

---

# 6. RÉSUMÉ DES FORMULES

| Type | Formule |
|------|---------|
| **A (Gastro)** | `max(3, ceil(Σ besoin_secretaires))` |
| **A (Dermato 1 méd)** | `max(3, ceil(Σ besoin_secretaires))` |
| **A (Dermato 2+ méd)** | `max(4, ceil(Σ besoin_secretaires))` |
| **B** | `max(1, ceil(Σ besoin_secretaires))` |
| **C (Bloc)** | Selon `types_intervention_besoins_personnel` |
| **D (Admin)** | Aucun besoin calculé |
| **Samedi** | `nombre_médecins` |

---

*Voir aussi : 1_DOMAIN_MODEL.md, 3_CONSTRAINTS.md, 4_OUTPUT.md*
