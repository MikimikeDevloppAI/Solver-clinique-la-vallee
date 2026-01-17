# Constraints and Scoring Rules

This document describes **all constraints** and scoring rules for the OptaPlanner solver.

---

# 1. SCORE STRUCTURE

## 1.1 HardMediumSoftScore

OptaPlanner uses **lexicographic comparison** on 3 levels:

```
Score = (Hard, Medium, Soft)

Comparison:
  (0, 0, -1000) > (-1, 0, +999999)   // Hard dominates everything
  (0, 0, -1000) > (0, -1, +999999)   // Medium dominates Soft
```

| Level | Role | Goal |
|-------|------|------|
| **HARD** | Feasibility | Must = 0 |
| **MEDIUM** | Coverage | Should = 0 |
| **SOFT** | Quality | Maximize |

## 1.2 Violation Degree Principle

Always specify the **degree** of violation to guide the solver:

```java
// BAD: no gradient
if (violation) → -100h

// GOOD: gradient for solver
for each violation → -100h
```

---

# 2. HARD CONSTRAINTS (Feasibility)

**Hard score < 0 = INVALID solution**

## 2.1 Complete List

| ID | Constraint | Rule | Penalty |
|----|------------|------|---------|
| H1 | **Time conflict** | 1 staff = 1 shift max per period | -100h per conflict |
| H2 | **Skill eligibility** | Staff must have the required skill | -100h per violation |
| H2b | **Site eligibility** | Staff must have the site in preferences | -100h per violation |
| H3 | **Surgical-Distant exclusion** | Surgical + Distant site same day = forbidden | -100h per violation |
| H4 | **Florence Bron 2F Tuesday** | Absolute prohibition | -100h |
| H5 | **Lucie Pratillo 2F/3F** | Absolute prohibition | -100h |
| H6 | **Exact days (flexible)** | Flexible staff = exactly `days_per_week` | -100h per day difference |
| H7 | **Absence** | Staff cannot be assigned during absence | -100h per violation |
| H8 | **Closing continuity** | 1R/2F = same person morning AND afternoon | -100h per violation |

## 2.2 Constraint Details

### H1 - Time Conflict

A staff member can only be assigned to **1 shift** per period.

```
Violation: Marie assigned to Ophtalmo AND Dermato Monday morning
Penalty: -100h
```

### H2 - Skill Eligibility

Staff must have the skill in their preferences to be assigned.

```
Violation: Pierre assigned to Accueil Dermato but doesn't have this skill
Penalty: -100h
```

**For Surgical Block**: Staff must have a preference for the required skill (e.g., Instrumentiste, Circulante).

```
Violation: Marie assigned as Instrumentiste but doesn't have this skill in preferences
Penalty: -100h
```

**Note**: Skills are unified - same eligibility check for consultation and surgical skills.

### H2b - Site Eligibility

Staff must have the site in preferences (P1, P2, P3 or P4) to be assigned.

```
Violation: Pierre assigned to Porrentruy but doesn't have this site in preferences
Penalty: -100h
```

**Note**: Staff without site preference for a location CANNOT be assigned there.

### H3 - Surgical-Distant Site Exclusion

**Forbidden** combinations on the same day:

| Morning | Afternoon | Forbidden |
|---------|-----------|-----------|
| Surgical block | Porrentruy | ✓ |
| Surgical block | Vieille Ville | ✓ |
| Porrentruy | Surgical block | ✓ |
| Vieille Ville | Surgical block | ✓ |

### H4 & H5 - Individual Rules

| Staff | ID | Prohibition |
|-------|-----|-------------|
| Florence Bron | `1e5339aa-5e82-4295-b918-e15a580b3396` | 2F on Tuesday |
| Lucie Pratillo | `5d3af9e3-674b-48d6-b54f-bd84c9eee670` | 2F and 3F always |

### H6 - Exact Days (Flexible Staff)

For `has_flexible_schedule = true`:

```
days_assigned MUST == days_per_week

Example:
  Marie: days_per_week = 4
  Assigned 3 days → -100h (1 day difference)
  Assigned 5 days → -100h (1 day difference)
```

### H7 - Absence

Staff with absence can **never** be assigned during that slot.

```
Marie: absence Wednesday (full day)
Violation: Marie assigned Wednesday morning
Penalty: -100h
```

### H8 - Closing Continuity

At a location with closing, if physicians present **morning AND afternoon**:

```
1R and 2F roles must be the SAME person all day

Example (VALID):
  Ophtalmo Monday morning: Marie (1R), Pierre (2F)
  Ophtalmo Monday PM: Marie (1R), Pierre (2F) ✓

Example (INVALID):
  Ophtalmo Monday morning: Marie (1R), Pierre (2F)
  Ophtalmo Monday PM: Anna (1R), Luc (2F) ✗ → -200h (2 violations)
```

---

# 3. MEDIUM CONSTRAINTS (Coverage)

**Medium score < 0 = unmet needs**

## 3.1 Complete List

| ID | Constraint | Rule | Penalty |
|----|------------|------|---------|
| M1 | **Missing skill** | Each required skill must be covered | **-1000m** per skill |
| M2 | **Uncovered closing** | Sites with closing missing 1R or 2F | **-1000m** per role |

## 3.2 Justification for -1000m

Medium penalty must be **greater than** worst Soft accumulation:

```
Worst Soft accumulation ≈ 810:
  - P234 day 5+: -200
  - Closing tier 4: -500
  - Cumulative: -50
  - Site change: -20
  - Lost preference: -40

With -1000m:
  Solver ALWAYS prefers filling a shift (even with -810 soft)
  rather than leaving it empty (-1000m) and putting staff in Admin (+15 soft)
```

## 3.3 Examples

### M1 - Missing Skill

```
Dermato Monday morning with 2 physicians:
  Need: 2 Aide Dermato + 2 Accueil Dermato = 4 staff
  Assigned: 2 Aide + 1 Accueil = 3 staff

  Violation: 1 Accueil Dermato missing
  Penalty: -1000m
```

### M2 - Uncovered Closing

```
Ophtalmo Monday (physicians morning + PM):
  Need: 1R + 2F
  Assigned: 1R only

  Violation: 2F missing
  Penalty: -1000m
```

---

# 4. SOFT CONSTRAINTS (Quality)

**Soft score = solution quality**

## 4.1 Preferences (3 Dimensions)

Take the **MAX** of 3 scores for each assignment.

| Dimension | P1 | P2 | P3 | P4 |
|-----------|-----|-----|-----|-----|
| **Skill** | +100 | +80 | +60 | - |
| **Physician** | +70 | +50 | - | - |
| **Site** | +40 | +35 | +30 | +25 |

```
Assignment_Score = max(Skill_Score, Physician_Score, Site_Score)
```

**Example**:
```
Marie assigned to Ophtalmo La Vallée with Dr. Martin:
  - Skill Accueil Ophtalmo: P1 → +100
  - Site La Vallée: P1 → +40
  - Physician Dr. Martin: P2 → +50

Score = max(100, 40, 50) = +100 soft
```

## 4.2 Site Continuity

| Situation | Score |
|-----------|-------|
| Same site morning + afternoon | **+20** |
| Site change | **-20** |
| Admin involved (either side) | **0** |

```
Marie: Ophtalmo morning + Ophtalmo PM → +20 soft
Pierre: Dermato morning + Porrentruy PM → -20 soft
Anna: Ophtalmo morning + Admin PM → 0 soft
```

## 4.3 Load Fairness

**Quadratic formula** to encourage balancing:

```
fairness_score -= Σ (load_r)²
```

**Why quadratic?**
```
Transfer 1 shift from Marie (5 shifts) to Pierre (2 shifts):
  Before: 5² + 2² = 29
  After: 4² + 3² = 25
  Improvement: +4 soft ✓
```

## 4.4 Distant Site Balancing (P234)

For staff whose site is P2/P3/P4:

| Day # at distant site | Penalty |
|-----------------------|---------|
| 1st | 0 |
| 2nd | -20 |
| 3rd | -50 |
| 4th | -100 |
| 5th+ | -200 |

```
Marie (Porrentruy = P3):
  Day 1: 0
  Day 2: -20
  Day 3: -50
  Total if 3 days: -70 soft
```

## 4.5 Closing Role Balancing

**Load score** = days_1R × 10 + days_2F × 12

| Score | Penalty |
|-------|---------|
| ≤ 22 | 0 |
| 23-29 | -30 |
| 30-31 | -80 |
| 32-35 | -150 |
| > 35 | -500 |

*Only the highest tier applies.*

```
Marie this week: 2 days 1R + 1 day 2F
Score = 2×10 + 1×12 = 32 → -150 soft
```

## 4.6 Admin Bonus

### Staff with `prefers_admin = true`

| Condition | Bonus |
|-----------|-------|
| Admin shift up to `admin_half_days_target` | +15 per shift |
| Admin shift beyond | +5 per shift |

### Staff with `prefers_admin = false`

Decreasing score: +10, +9, +8, +7...

```
Pierre (prefers_admin = false), 3 admin shifts:
  Shift 1: +10
  Shift 2: +9
  Shift 3: +8
  Total: +27 soft
```

## 4.7 P234 + Closing Cumulative

If a staff member has **simultaneously**:
- Active P234 penalty (≥2 days at distant site)
- Active closing penalty (score > 22)

**Additional penalty**: -50 soft

## 4.8 Admin Preference for Closing Role (Half-day)

If a site with closing has physicians only **morning OR afternoon**:

| Situation | Bonus |
|-----------|-------|
| Staff 1R/2F in morning → Admin in afternoon | +30 |
| Staff 1R/2F in afternoon → Admin in morning | +30 |

---

# 5. SPECIAL RULES

## 5.1 Paul Jacquier and 3F Role

| Staff | ID |
|-------|-----|
| Paul Jacquier | `121dc7d9-99dc-46bd-9b6c-d240ac6dc6c8` |

**Rule**: If Paul Jacquier is present **Thursday AND Friday** at a location with closing, a **3F** role is required.

## 5.2 Physician-Staff Bonus

| Physician | Staff | Bonus |
|-----------|-------|-------|
| Dr. fda323f4 (`fda323f4-3efd-4c78-8b63-7d660fcd7eea`) | Sara Bortolon, Mirlinda Hasani | +50 soft |

| Staff | ID |
|-------|-----|
| Sara Bortolon | `68e74e31-12a7-4fd3-836d-41e8abf57792` |
| Mirlinda Hasani | `324639fa-2e3d-4903-a143-323a17b0d988` |

## 5.3 Obstetrician Alone at Ophtalmo Site

If an Ophtalmo site has **no standard physicians** but has **≥1 obstetrician**:

```
Demand = 1 Accueil Ophtalmo (instead of 0)
```

---

# 6. SCORE SUMMARY

## 6.1 Hard Constraints (-100h each)

- H1: Time conflict
- H2: Skill eligibility
- H2b: Site eligibility
- H3: Surgical-Distant exclusion
- H4: Florence Bron 2F Tuesday
- H5: Lucie Pratillo 2F/3F
- H6: Exact days (flexible)
- H7: Absence
- H8: Closing continuity

## 6.2 Medium Constraints

| Constraint | Penalty |
|------------|---------|
| Missing skill | -1000m |
| Missing closing role | -1000m |

## 6.3 Soft Constraints - Bonuses

| Element | Score |
|---------|-------|
| Skill P1 | +100 |
| Skill P2 | +80 |
| Skill P3 | +60 |
| Physician P1 | +70 |
| Physician P2 | +50 |
| Site P1 | +40 |
| Site P2 | +35 |
| Site P3 | +30 |
| Site P4 | +25 |
| Site continuity | +20 |
| Admin (preferred) | +15 |
| Admin (not preferred) | +10 decreasing |
| Admin if closing role half-day | +30 |
| Dr. fda323f4 bonus | +50 |

## 6.4 Soft Constraints - Penalties

| Element | Score |
|---------|-------|
| Site change | -20 |
| P234 day 2 | -20 |
| P234 day 3 | -50 |
| P234 day 4 | -100 |
| P234 day 5+ | -200 |
| Closing tier 1 (23-29) | -30 |
| Closing tier 2 (30-31) | -80 |
| Closing tier 3 (32-35) | -150 |
| Closing tier 4 (>35) | -500 |
| P234 + closing cumulative | -50 |
| Fairness | -Σ(load²) |

---

# 7. GLOBAL FORMULA

```
Score = HardMediumSoftScore(

   hard = Σ violations_H1_to_H8 × (-100),

   medium = Σ missing_skills × (-1000)
          + Σ missing_closing_roles × (-1000),

   soft = Σ max(skill_score, physician_score, site_score)  // Preferences
        + Σ site_continuity_bonus                           // +20 if same site
        + Σ admin_bonus                                     // +15 or decreasing
        + Σ closing_admin_bonus                             // +30 if half-day
        + Σ special_physician_bonus                         // +50 Dr. fda323f4
        - Σ site_change_penalties                           // -20
        - Σ P234_penalties                                  // -20 to -200
        - Σ closing_penalties                               // -30 to -500
        - Σ cumulative_penalties                            // -50
        - Σ (load_r)²                                       // Quadratic fairness
)
```

---

*OptaPlanner constraint specification document*
*See also: 1_DOMAIN_MODEL.md, 2_NEEDS_CALCULATION.md, 4_OUTPUT.md*
