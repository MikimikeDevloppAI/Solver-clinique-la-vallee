# Staff Needs Calculation

This document explains how to calculate staff needs from the physician schedule.

---

## Overview

```
INPUT                               OUTPUT
=====                               ======
assignments (type='physician') →    Needs per (location, date, period)
surgical_procedures            →      ├── number of staff
                                      ├── required skills
                                      └── closing roles
```

---

# 1. CALCULATION FLOW

## Step 1: Read physician schedule

```
SELECT * FROM assignments
WHERE type = 'physician'
  AND date BETWEEN start_date AND end_date
```

## Step 2: Group by (location, date, period)

For each group, list physicians present with their `staff_requirement`.

## Step 3: Calculate raw need

```
raw_need = Σ physicians.staff_requirement
```

**Exception**: Obstetricians (`is_obstetrician = true`) have `staff_requirement = 0` and don't count.

## Step 4: Apply rules by staffing type

See section 2.

## Step 5: Add closing needs

See section 3.

---

# 2. RULES BY STAFFING TYPE

## Type A: Fixed Ratio (Gastro, Dermato)

### Gastro (Vieille Ville)

| Condition | Aide de salle Gastro | Accueil Gastro | Total |
|-----------|---------------------|----------------|-------|
| ≥1 physician | 2 | 1 | **3 minimum** |

### Dermato (La Vallée)

| Condition | Aide de salle Dermato | Accueil Dermato | Total |
|-----------|----------------------|-----------------|-------|
| 1 physician | 2 | 1 | **3 minimum** |
| ≥2 physicians | 2 | 2 | **4 minimum** |

**Final formula**: `max(ceil(raw_need), type_A_minimum)`

---

## Type B: Standard Formula (Ophtalmo, Angio, Rhuma, ORL)

| Condition | Demand |
|-----------|--------|
| 0 physicians | 0 |
| ≥1 physician | `max(1, ceil(raw_need))` |

**Skill**: Reception skill for the specialty (e.g., Accueil Ophtalmo)

---

## Type C: Surgical Block

Calculation is different: based on scheduled **procedures**.

### Flow

```
1. Read procedures for the day
2. For each procedure:
   - Get procedure_type_id
   - Look up in procedure_type_skills
   - Create a need for each required skill
3. Aggregate needs by (date, period, skill)
```

### Example

```
Surgical Block - Monday morning
===============================

Scheduled procedures:
  - Red Room: Cataract Surgery
  - Gastro Room: Digestive Endoscopy

Needs per procedure (from procedure_type_skills):
  Cataract     → 1 Instrumentiste + 1 Circulante
  Endoscopy    → 1 Aide bloc gastro + 1 Circulante

Total needs:
  - 1 × Instrumentiste
  - 1 × Aide bloc gastro
  - 2 × Circulante (or 1 if same person covers both rooms)
```

---

## Type D: Admin

- No need calculation
- Unlimited capacity
- No skill required
- Any staff member can be assigned

---

# 3. CLOSING NEEDS

For locations with `has_closing = true` (Ophtalmo, Porrentruy).

## Rule

If physicians are present at a location with closing:
- Role **1R** (First Responsibility) required
- Role **2F** (Second Closing) required

## Full day vs half-day

| Situation | Rule |
|-----------|------|
| Physicians morning AND afternoon | 1R and 2F must be the **SAME person** all day |
| Physicians morning only | 1R and 2F in morning, bonus if Admin in afternoon |
| Physicians afternoon only | 1R and 2F in afternoon, bonus if Admin in morning |

## Special case: Paul Jacquier

If Paul Jacquier (`121dc7d9-99dc-46bd-9b6c-d240ac6dc6c8`) is present **Thursday AND Friday** at a location with closing:
- Role **3F** (Third Closing) required in addition

---

# 4. EXCEPTIONS

## Saturday

```
need = number_of_physicians (no ceiling, no minimum)
```

## Obstetricians alone at Ophtalmo

If an Ophtalmo location has **no standard physicians** but has **≥1 obstetrician**:

```
need = 1 Accueil Ophtalmo
```

---

# 5. COMPLETE EXAMPLES

## Example 1: Dermato with 2 physicians

```
Dermato - Monday January 5 - Morning
====================================

INPUT (assignments WHERE type='physician'):
  - Dr. Martin (staff_requirement = 1.5)
  - Dr. Dupont (staff_requirement = 1.8)

CALCULATION:
  1. raw_need = 1.5 + 1.8 = 3.3
  2. ceil(3.3) = 4
  3. Dermato with 2+ physicians → minimum 4
  4. max(4, 4) = 4 staff members

GENERATED NEEDS:
  - 2 × Aide de salle Dermato (skill_type: consultation)
  - 2 × Accueil Dermato (skill_type: consultation)
```

## Example 2: Ophtalmo with closing

```
Ophtalmo - Tuesday January 6
============================

Morning:
  - Dr. Bernard (staff_requirement = 1.2)
  - Dr. Thomas (staff_requirement = 1.2)

Afternoon:
  - Dr. Bernard (staff_requirement = 1.2)

CALCULATION:
  Morning: ceil(2.4) = 3 Accueil Ophtalmo
  Afternoon: ceil(1.2) = 2 Accueil Ophtalmo

CLOSING (physicians morning AND PM):
  - 1R required (same person all day)
  - 2F required (same person all day)

GENERATED NEEDS:
  Morning: 3 × Accueil Ophtalmo + 1R + 2F
  Afternoon: 2 × Accueil Ophtalmo + 1R + 2F (same persons)
```

## Example 3: Surgical block

```
Surgical Block - Wednesday January 7 - Morning
==============================================

PROCEDURES (from surgical_procedures):
  - 08:00 Red Room: General Surgery (Dr. Lefevre)
  - 08:30 Gastro Room: Endoscopy (Dr. Dubois)
  - 10:00 Green Room: Cataract (Dr. Martin)

NEEDS PER PROCEDURE (from procedure_type_skills):
  General Surgery → 1 Instrumentiste + 1 Circulante + 1 Aide opératoire
  Endoscopy → 1 Aide bloc gastro + 1 Circulante
  Cataract → 1 Instrumentiste + 1 Circulante

AGGREGATION:
  - 2 × Instrumentiste (Red + Green in parallel)
  - 1 × Aide bloc gastro
  - 1 × Aide opératoire
  - 3 × Circulante (or less if one person can cover multiple rooms)
```

## Example 4: Saturday

```
Ophtalmo - Saturday January 10 - Morning
========================================

INPUT:
  - Dr. Martin (staff_requirement = 1.2)
  - Dr. Dupont (staff_requirement = 1.5)

SATURDAY CALCULATION:
  need = 2 (number of physicians, not ceil(2.7))

GENERATED NEEDS:
  - 2 × Accueil Ophtalmo
```

---

# 6. FORMULA SUMMARY

| Type | Formula |
|------|---------|
| **A (Gastro)** | `max(3, ceil(Σ staff_requirement))` |
| **A (Dermato 1 phys)** | `max(3, ceil(Σ staff_requirement))` |
| **A (Dermato 2+ phys)** | `max(4, ceil(Σ staff_requirement))` |
| **B** | `max(1, ceil(Σ staff_requirement))` |
| **C (Surgical)** | Based on `procedure_type_skills` |
| **D (Admin)** | No calculated need |
| **Saturday** | `number_of_physicians` |

---

*See also: 1_DOMAIN_MODEL.md, 3_CONSTRAINTS.md, 4_OUTPUT.md*
