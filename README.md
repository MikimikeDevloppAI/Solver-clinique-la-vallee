# Solver Clinique La Vallée

Solveur OptaPlanner pour la planification des secrétaires médicales de la Clinique La Vallée.

## Problème

**Employee Shift Scheduling** : Affecter des secrétaires médicales à des shifts (demi-journées) sur différents emplacements, en respectant les compétences requises et les préférences.

## Architecture du Modèle

```
SITE (géographique)
  └── EMPLACEMENT (service)
        └── SPÉCIALITÉ
              └── SKILLS requis
```

### Sites
| Site | Caractéristique |
|------|-----------------|
| Clinique La Vallée | Site principal (Delémont) |
| Porrentruy | Site distant |
| Vieille Ville Delémont | Site Gastro |

### Types d'Emplacements
| Type | Exemples | Règle de staffing |
|------|----------|-------------------|
| **Type A** | Gastro, Dermato | Ratios fixes Aide/Accueil |
| **Type B** | Ophtalmo, Angio, Rhuma, ORL | Skill unique, formule standard |
| **Type C** | Bloc opératoire | Skills par intervention |
| **Type D** | Admin | Aucun skill requis |

## Score OptaPlanner

**HardMediumSoftScore** avec comparaison lexicographique :

```
HARD > MEDIUM > SOFT

(0h, 0m, -1000s) est MEILLEUR que (-1h, 0m, +999999s)
```

| Niveau | Rôle | Objectif |
|--------|------|----------|
| **HARD** | Faisabilité | = 0 obligatoire |
| **MEDIUM** | Couverture | = 0 souhaitable |
| **SOFT** | Qualité | Maximiser |

## Deux Types de Ressources

| Type | Input | Solveur décide |
|------|-------|----------------|
| **Flexible** | `nombre_jours_semaine` + `absences` | Quels jours + où |
| **Fixe** | `shifts_fixes` (créneaux prédéfinis) | Seulement où |

## Documentation

| Document | Description |
|----------|-------------|
| [docs/INPUT_SCHEMA.md](docs/INPUT_SCHEMA.md) | Structure des données d'entrée |
| [docs/OUTPUT_SCHEMA.md](docs/OUTPUT_SCHEMA.md) | Structure des données de sortie |
| [docs/CONSTRAINTS.md](docs/CONSTRAINTS.md) | Règles et contraintes de scoring |

## Contraintes Principales

### Hard (8 contraintes, -100h chacune)
- Conflit temporel
- Éligibilité skill
- Exclusion Bloc-Site distant
- Jours exacts (ressources flexibles)
- Absences
- Continuité fermeture
- Règles individuelles (Florence Bron, Lucie Pratillo)

### Medium (-1000m)
- Skill manquant
- Rôle fermeture manquant

### Soft
- Préférences : Skill (P1=+100) > Médecin (P1=+70) > Site (P1=+40)
- Continuité site : +20
- Équité quadratique : -Σ(charge²)
- Équilibrage P234 et fermeture

## Configuration Recommandée

| Paramètre | Valeur |
|-----------|--------|
| Type de score | HardMediumSoftScore |
| Limite de temps | 5 minutes (300 000 ms) |
| Algorithme | Late Acceptance ou Tabu Search |

## Licence

Propriétaire - Clinique La Vallée
