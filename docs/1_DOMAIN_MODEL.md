# Domain Model - Medical Staff Scheduling

This document describes the data model for the OptaPlanner solver.

---

## Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA MODEL                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  GEOGRAPHIC STRUCTURE                                           │
│  ====================                                           │
│  sites                                                          │
│    └── locations                                                │
│          └── specialties                                        │
│                └── skills (unified)                             │
│                                                                  │
│  PEOPLE                                                         │
│  ======                                                         │
│  physicians (staff_requirement, specialty)                      │
│  staff_members (preferences, availability)                      │
│                                                                  │
│  UNIFIED SCHEDULE                                               │
│  ================                                               │
│  assignments                                                    │
│    ├── type = 'physician' | 'staff'                             │
│    ├── person_id                                                │
│    ├── location_id, date, period                                │
│    └── skill_id, procedure_id (context-dependent)               │
│                                                                  │
│  SURGICAL BLOCK                                                 │
│  ==============                                                 │
│  surgical_procedures                                            │
│  procedure_types → procedure_type_skills                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Key concept**: Skills are unified - consultation skills (Accueil Ophtalmo) and surgical roles (Instrumentiste) are ALL entries in the same `skills` table.

---

# 1. GEOGRAPHIC STRUCTURE

## 1.1 Table: `sites`

Geographic places where staff work.

| id | name | distance_type |
|----|------|---------------|
| `site-la-vallee` | Clinique La Vallée | reference (main site) |
| `site-porrentruy` | Porrentruy | distant |
| `site-vieille-ville` | Vieille Ville Delémont | nearby |

**Fields**: `id`, `name`, `address`, `distance_type`

---

## 1.2 Table: `locations`

Departments/services within a site.

| id | name | site | staffing_type | has_closing |
|----|------|------|---------------|-------------|
| `7c8abe96-...` | Ophtalmologie | La Vallée | B | ✓ |
| `d82c55ee-...` | Dermatologie | La Vallée | A | ✗ |
| `c3bd13...` | Angiologie | La Vallée | B | ✗ |
| `0e1b31...` | Rhumatologie | La Vallée | B | ✗ |
| `86f1047f-...` | Bloc opératoire | La Vallée | C | ✗ |
| `7723c334-...` | Gastroentérologie | Vieille Ville | A | ✗ |
| `043899a1-...` | Porrentruy | Porrentruy | B | ✓ |
| `00000000-...0001` | Administratif | - | D | ✗ |

**Fields**: `id`, `name`, `site_id`, `staffing_type`, `specialty_id`, `has_closing`

### Staffing Types

| Type | Description | Locations | Characteristic |
|------|-------------|-----------|----------------|
| **A** | Fixed Aide/Accueil ratio | Gastro, Dermato | Minimum per skill based on physician count |
| **B** | Standard formula | Ophtalmo, Angio, Rhuma, ORL | Single skill, ceil(Σ requirement) |
| **C** | Skills per procedure | Surgical block | Needs defined by procedure type |
| **D** | No skill required | Admin | Unlimited capacity, open to all |

---

## 1.3 Table: `specialties`

Medical specialties.

**Fields**: `id`, `name`, `code`

---

## 1.4 Table: `skills` (UNIFIED)

**All competencies are skills** - whether for consultations, surgical block, or closing roles.

| id | name | skill_type | specialty_id |
|----|------|------------|--------------|
| `skill-accueil-ophtalmo` | Accueil Ophtalmo | consultation | ophtalmo |
| `skill-accueil-gastro` | Accueil Gastro | consultation | gastro |
| `skill-aide-gastro` | Aide de salle Gastro | consultation | gastro |
| `skill-accueil-dermato` | Accueil Dermato | consultation | dermato |
| `skill-aide-dermato` | Aide de salle Dermato | consultation | dermato |
| `skill-instrumentiste` | Instrumentiste | surgical | - |
| `skill-circulante` | Circulante | surgical | - |
| `skill-aide-operatoire` | Aide opératoire | surgical | - |
| `skill-aide-bloc-gastro` | Aide bloc gastro | surgical | gastro |
| `skill-1r` | 1R (Première Responsabilité) | closing | - |
| `skill-2f` | 2F (Deuxième Fermeture) | closing | - |
| `skill-3f` | 3F (Troisième Fermeture) | closing | - |

**Fields**: `id`, `name`, `skill_type`, `specialty_id`

### Skill Types

| skill_type | Description |
|------------|-------------|
| `consultation` | Skills for consultation locations (Accueil X, Aide de salle X) |
| `surgical` | Skills for surgical block (Instrumentiste, Circulante, etc.) |
| `closing` | Closing responsibility roles (1R, 2F, 3F) |

---

# 2. PEOPLE

## 2.1 Table: `physicians`

**Fields**: `id`, `first_name`, `last_name`, `specialty_id`, `staff_requirement`, `is_obstetrician`

| Field | Description |
|-------|-------------|
| `staff_requirement` | Requirement coefficient (e.g., 1.0, 1.2, 1.5, 2.0) |
| `is_obstetrician` | Exception: requirement = 0, not counted in calculation |

---

## 2.2 Table: `staff_members`

**Fields**: `id`, `first_name`, `last_name`, `is_active`, `has_flexible_schedule`, `days_per_week`, `prefers_admin`, `admin_half_days_target`

### Schedule Types

| Type | `has_flexible_schedule` | Data | Solver decides |
|------|------------------------|------|----------------|
| **Flexible** | `true` | `days_per_week` + absences | Which days + where |
| **Fixed** | `false` | Predefined shifts in assignments | Only where |

---

## 2.3 Preference Tables

Preferences determine **eligibility** AND **scoring**.

| Table | Fields | Eligibility |
|-------|--------|-------------|
| `staff_site_preferences` | staff_id, site_id, priority (P1-P4) | No preference = **not eligible** |
| `staff_skill_preferences` | staff_id, skill_id, priority (P1-P3) | No preference = **not eligible** |
| `staff_physician_preferences` | staff_id, physician_id, priority (P1-P2) | Optional (bonus) |

**Note**: Since skills are unified, `staff_skill_preferences` covers both consultation skills AND surgical skills.

---

## 2.4 Staff with Special Rules

| id | Name | Rule |
|----|------|------|
| `1e5339aa-...` | Florence Bron | No 2F on Tuesday |
| `5d3af9e3-...` | Lucie Pratillo | Never 2F or 3F |
| `121dc7d9-...` | Paul Jacquier | Triggers 3F requirement if present Thu+Fri |
| `68e74e31-...` | Sara Bortolon | Bonus with Dr. fda323f4 |
| `324639fa-...` | Mirlinda Hasani | Bonus with Dr. fda323f4 |

---

# 3. UNIFIED SCHEDULE

## 3.1 Table: `assignments`

Single table containing schedules for both physicians AND staff.

**Fields**:
- `id`, `date`, `period` (`morning` | `afternoon`)
- `location_id`
- **`type`**: `physician` | `staff`
- **`person_id`**: UUID of physician or staff member
- `skill_id` (for staff in consultations OR surgical block)
- `procedure_id` (for surgical block, links to specific procedure)
- `is_1r`, `is_2f`, `is_3f` (closing roles)

### Usage

| type | Role |
|------|------|
| `physician` | **INPUT** - Physician schedule (source of needs) |
| `staff` | **OUTPUT** - Assignments generated by solver |

---

# 4. SURGICAL BLOCK

## 4.1 Table: `surgical_procedures`

Schedule of surgical interventions.

**Fields**: `id`, `date`, `period`, `procedure_type_id`, `physician_id`, `operating_room`

### Operating Rooms

| Room | Procedure Types |
|------|-----------------|
| red_room | General surgery, Cataract |
| green_room | General surgery, Angiology |
| yellow_room | General surgery |
| gastro_room | Digestive endoscopies |

---

## 4.2 Table: `procedure_types`

Definition of surgical procedures.

**Fields**: `id`, `name`, `code`, `specialty_id`

---

## 4.3 Table: `procedure_type_skills`

Each procedure type defines required skills (personnel needs).

**Fields**: `procedure_type_id`, `skill_id`, `quantity`

**Example**:

| Procedure Type | Skill | Quantity |
|----------------|-------|----------|
| Cataract Surgery | Instrumentiste | 1 |
| Cataract Surgery | Circulante | 1 |
| Digestive Endoscopy | Aide bloc gastro | 1 |
| Digestive Endoscopy | Circulante | 1 |
| General Surgery | Instrumentiste | 1 |
| General Surgery | Circulante | 1 |
| General Surgery | Aide opératoire | 1 |

**Note**: Skills here reference the unified `skills` table (skill_type = 'surgical').

---

# 5. AVAILABILITY

## 5.1 Table: `staff_absences`

For staff with `has_flexible_schedule = true`.

**Fields**: `staff_id`, `date`, `period` (`morning` | `afternoon` | null = full day)

**Example**:
```
Marie (flexible, 4 days/week):
  - Absence Wednesday (full day)
  - Absence Friday morning

→ Available: Monday, Tuesday, Thursday, Friday PM, Saturday
→ Solver chooses 4 days from these
```

## 5.2 Fixed Shifts

For staff with `has_flexible_schedule = false`, shifts are predefined in the assignments table.

**Example**:
```
Pierre (fixed) - entries in assignments (type='staff', location=null):
  - Monday morning
  - Monday afternoon
  - Tuesday morning
  - Thursday afternoon

→ Solver MUST assign Pierre on these 4 slots
→ Solver chooses the location for each slot
```

---

# 6. REFERENCE IDs

## Locations

| ID | Name |
|----|------|
| `00000000-0000-0000-0000-000000000001` | Administratif |
| `7723c334-d06c-413d-96f0-be281d76520d` | Vieille Ville (Gastro) |
| `7c8abe96-0a6b-44eb-857f-ad69036ebc88` | Ophtalmologie |
| `d82c55ee-2964-49d4-a578-417b55b557ec` | Dermatologie |
| `043899a1-a232-4c4b-9d7d-0eb44dad00ad` | Porrentruy |
| `86f1047f-c4ff-441f-a064-42ee2f8ef37a` | Bloc opératoire |

---

# 7. TABLE SUMMARY

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

---

*See also: 2_NEEDS_CALCULATION.md, 3_CONSTRAINTS.md, 4_OUTPUT.md*
