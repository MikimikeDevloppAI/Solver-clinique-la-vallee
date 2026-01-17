# Medical Staff Scheduler - Clinique La Vallée

OptaPlanner solver for medical staff scheduling at Clinique La Vallée.

## Problem

**Employee Shift Scheduling**: Assign medical staff to shifts (half-days) at different locations, respecting required skills and preferences.

## Model Architecture

```
sites (geographic)
  └── locations (departments)
        └── specialties
              └── skills (unified)
```

### Sites
| Site | Characteristic |
|------|----------------|
| Clinique La Vallée | Main site (Delémont) |
| Porrentruy | Distant site |
| Vieille Ville Delémont | Gastro site |

### Location Types (Staffing)
| Type | Examples | Staffing Rule |
|------|----------|---------------|
| **Type A** | Gastro, Dermato | Fixed Aide/Accueil ratios |
| **Type B** | Ophtalmo, Angio, Rhuma, ORL | Single skill, standard formula |
| **Type C** | Surgical block | Skills per procedure |
| **Type D** | Admin | No skill required |

## Key Concept: Unified Skills

All competencies are stored in a single `skills` table:
- **Consultation skills**: Accueil Ophtalmo, Aide de salle Gastro, etc.
- **Surgical skills**: Instrumentiste, Circulante, Aide opératoire, etc.
- **Closing roles**: 1R, 2F, 3F

## OptaPlanner Score

**HardMediumSoftScore** with lexicographic comparison:

```
HARD > MEDIUM > SOFT

(0h, 0m, -1000s) is BETTER than (-1h, 0m, +999999s)
```

| Level | Role | Goal |
|-------|------|------|
| **HARD** | Feasibility | Must = 0 |
| **MEDIUM** | Coverage | Should = 0 |
| **SOFT** | Quality | Maximize |

## Two Types of Staff Schedules

| Type | Input | Solver decides |
|------|-------|----------------|
| **Flexible** | `days_per_week` + `absences` | Which days + where |
| **Fixed** | predefined shifts | Only where |

## Documentation

| Document | Description |
|----------|-------------|
| [docs/1_DOMAIN_MODEL.md](docs/1_DOMAIN_MODEL.md) | Data model (tables, relationships) |
| [docs/2_NEEDS_CALCULATION.md](docs/2_NEEDS_CALCULATION.md) | Staff needs calculation |
| [docs/3_CONSTRAINTS.md](docs/3_CONSTRAINTS.md) | Scoring rules and constraints |
| [docs/4_OUTPUT.md](docs/4_OUTPUT.md) | Solver output format |

## Main Constraints

### Hard (9 constraints, -100h each)
- Time conflict
- Skill eligibility
- Site eligibility
- Surgical-Distant site exclusion
- Exact days (flexible staff)
- Absences
- Closing continuity
- Individual rules (Florence Bron, Lucie Pratillo)

### Medium (-1000m)
- Missing skill
- Missing closing role

### Soft
- Preferences: Skill (P1=+100) > Physician (P1=+70) > Site (P1=+40)
- Site continuity: +20
- Quadratic fairness: -Σ(load²)
- P234 and closing balancing

## Recommended Configuration

| Parameter | Value |
|-----------|-------|
| Score type | HardMediumSoftScore |
| Time limit | 5 minutes (300,000 ms) |
| Algorithm | Late Acceptance or Tabu Search |

## Table Summary

| Table | Description |
|-------|-------------|
| `sites` | Geographic locations |
| `locations` | Departments within sites |
| `specialties` | Medical specialties |
| `skills` | **Unified** competencies (consultation + surgical + closing) |
| `physicians` | Doctors with staff requirements |
| `staff_members` | Medical secretaries/staff |
| `staff_site_preferences` | Site eligibility & priority |
| `staff_skill_preferences` | Skill eligibility & priority |
| `staff_physician_preferences` | Optional physician bonuses |
| `assignments` | **Unified** schedule (physicians + staff) |
| `surgical_procedures` | Scheduled surgeries |
| `procedure_types` | Surgery type definitions |
| `procedure_type_skills` | Skills required per procedure |
| `staff_absences` | Unavailability periods |

## License

Proprietary - Clinique La Vallée
