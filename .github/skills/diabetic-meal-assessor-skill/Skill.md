[diabetic-meal-assessor-SKILL.md](https://github.com/user-attachments/files/32283229/diabetic-meal-assessor-SKILL.md)
---
name: diabetic-meal-assessor
description: >-
  Decide whether a dish is nutritionally suitable for a diabetic individual, judged from the
  dish's macronutrient contribution to total energy and its fibre content. Use it whenever the
  user asks if a dish/meal is suitable, safe, good, or OK for diabetics, is "diabetic-friendly",
  is fine for someone with diabetes/high blood sugar, or is appropriate from a macro/low-GI
  standpoint — whether the dish is described in chat, shown in an uploaded image, or listed in a
  PDF menu/recipe. This skill first calls the `nutrition-calculator` skill to get the per-100 g
  breakdown (Energy kcal, Carbohydrate, Protein, Fat, Fibre) and the ingredient fat ledger, then
  applies fixed carbohydrate, protein, and fat criteria to return a PASS/FAIL per criterion and a
  final "Suitable" / "Not suitable" verdict. Trigger even when the user doesn't say "macros",
  "Atwater", or "per 100 g" but clearly wants to know if a food works for a diabetic.
---

# Diabetic Meal Assessor

Given a dish, decide whether it is **nutritionally suitable for a diabetic individual**. The
judgement is made entirely from the dish's **per-100 g macronutrient profile** — specifically
how much of its energy comes from carbohydrate, protein, and fat (via standard Atwater factors),
plus its **fibre** content and the **source of its fat**. This skill does **not** estimate
nutrition itself; it consumes the output of the `nutrition-calculator` skill and applies a fixed
set of pass/fail rules on top of it.

This is dietary screening of a representative preparation, not personalised medical or clinical
advice. Restaurant portions and recipes vary, and individual carbohydrate tolerance differs — say
so in the output.

## Inputs this skill handles

The dish may arrive as a plain chat description ("is butter chicken OK for a diabetic?"), an
**image** of a menu or plate, or a **PDF** menu/recipe. You do not parse these yourself for
nutrition — you hand them to `nutrition-calculator` (see Step 1), which already knows how to read
descriptions, images, PDFs, and menus in any language. If the user names several dishes, run the
whole workflow for each dish separately.

---

## The procedure

Follow these five steps in order. Step 1 gets the numbers; Steps 2–4 are the assessment; Step 5
is the output.

### Step 1 — Get the per-100 g nutrition from `nutrition-calculator` (do not reimplement it)

**Always call the `nutrition-calculator` skill first** and use its result as the input to this
assessment. Pass along exactly what the user gave you (the dish description, image, or PDF).
Do **not** estimate energy, macros, fibre, or ingredient amounts yourself — that logic (authentic
recipe choice, IFCT lookup, cooking yield, fibre split) lives in `nutrition-calculator` and must
not be duplicated or second-guessed here.

From its output you need **two things**:

1. **The per-100 g table** — Energy (kcal), Carbohydrate (g), Protein (g), Fat (g), and Dietary
   fibre (g, total). These five numbers drive Criteria 1–3.
2. **The ingredient ledger** — specifically the **Fat (g) column per ingredient** for the whole
   recipe. You need this for the fat-source check in Criterion 3. `nutrition-calculator` produces
   this ledger in its Step 3; ask for / read the working, not just the summary table. If only the
   per-100 g summary is available, request the ingredient-level fat breakdown before assessing
   fat, because the fat criterion can't be completed without knowing which ingredients the fat
   comes from.

Restate the per-100 g figures you received (with the recipe basis `nutrition-calculator` used) so
the assessment is auditable.

### Step 2 — Convert macros to % of total energy (Atwater)

Using **standard Atwater factors**, convert each macronutrient's grams (per 100 g of dish) to
the energy it supplies:

```
Carbohydrate energy (kcal) = Carbohydrate g × 4
Protein energy (kcal)      = Protein g      × 4
Fat energy (kcal)          = Fat g          × 9
```

Then compute the **Atwater total energy** as the sum of those three, and each macronutrient's
percentage of it:

```
Atwater total (kcal) = Carb energy + Protein energy + Fat energy

Carb %    = Carb energy    ÷ Atwater total × 100
Protein % = Protein energy ÷ Atwater total × 100
Fat %     = Fat energy     ÷ Atwater total × 100
```

**Use the Atwater total (Carb×4 + Protein×4 + Fat×9) as the denominator**, not the Energy kcal
figure from the table. The two are usually within a few kcal, but dividing by the Atwater sum
makes the three percentages add up to exactly 100 % and keeps the criteria internally consistent.
Use the Carbohydrate value **as reported** by `nutrition-calculator` (do not subtract fibre from
it). **Show this calculation** — the three energy figures, the total, and the three percentages
(one decimal place) — as a small table.

### Step 3 — Apply the three criteria

Evaluate all three. Each is independent; note the exact numbers and which branch you took.

**Criterion 1 — Carbohydrate**
- **PASS** if Carb % is **strictly less than 55 %** of total energy.
- **FAIL** otherwise (Carb % ≥ 55 %).

**Criterion 2 — Protein**
- **PASS** if Protein % is **20 % or more** of total energy.
- If Protein % is **less than 20 %**, fall back to **fibre**:
  - **PASS** if Dietary fibre (total) is **at least 2 g per 100 g** of the dish.
  - **FAIL** otherwise (protein < 20 % **and** fibre < 2 g/100 g).

**Criterion 3 — Fat**
- **PASS** if Fat % is **strictly less than 30 %** of total energy.
- If Fat % is **30 % or more**, check the **source of the fat**:
  - **PASS** if **50 % or more of the total fat grams** in the dish come from one or more of these
    **approved fat sources**:

    > **avocado, avocado oil, nuts, seeds, mustard oil, sesame oil, groundnut oil, olive, olive
    > oil, or fish.**

  - **FAIL** otherwise (fat ≥ 30 % **and** less than half the fat is from approved sources).

**How to do the fat-source check.** Use the ingredient fat ledger from Step 1. For each
ingredient contributing fat, decide whether its fat is from an **approved** source (the list
above — e.g. peanuts/almonds/cashews and other **nuts**; sunflower/sesame/flax and other
**seeds**; **mustard / sesame / groundnut / olive** oils; **avocado**; **fish** and its own fat)
or a **non-approved** source (e.g. **butter, ghee, cream, coconut oil, refined/vegetable oil,
palm oil, cheese/dairy fat, meat/poultry fat, lard**). Sum the fat grams from approved sources and
divide by the total fat grams:

```
Approved-fat share = (sum of fat g from approved sources) ÷ (total fat g) × 100
```

PASS the source check if this share is **≥ 50 %**. **Show the breakdown explicitly** — list each
fat-contributing ingredient, its fat grams, and whether it's approved — so the estimate is
auditable. If an ingredient's classification is genuinely ambiguous (e.g. an unspecified "cooking
oil"), state your assumption and, when it would flip the verdict, ask the user which oil/fat the
dish actually uses.

### Step 4 — Overall verdict

- The dish is **"Suitable for diabetic individuals"** only if **all three criteria PASS**.
- If **any one** criterion FAILS, the dish is **"Not suitable"**. Name **which** criterion or
  criteria failed and give the one-line reason (the specific number that breached the threshold).

### Step 5 — Output the assessment

Present the result for each dish using this structure:

```markdown
## [Dish name] — diabetic suitability

**Per 100 g (from nutrition-calculator):**

| Energy (kcal) | Carbohydrate (g) | Protein (g) | Fat (g) | Dietary fibre (g) |
|--------------:|-----------------:|------------:|--------:|------------------:|
|          [E]  |            [C]    |       [P]   |   [F]   |            [Fib]  |

**Energy contribution (Atwater):**

| Macronutrient | Energy (kcal)      | % of total energy |
|---------------|-------------------:|------------------:|
| Carbohydrate  | [C×4]              | [Carb %]          |
| Protein       | [P×4]              | [Protein %]       |
| Fat           | [F×9]              | [Fat %]           |
| **Total**     | **[Atwater total]**| **100 %**         |

**Criteria:**

- **Carbohydrate — [PASS/FAIL].** Carb energy is [Carb %] (threshold: < 55 %).
- **Protein — [PASS/FAIL].** Protein energy is [Protein %] (threshold: ≥ 20 %).
  [If < 20 %: fibre check — fibre is [Fib] g/100 g (threshold: ≥ 2 g) → PASS/FAIL.]
- **Fat — [PASS/FAIL].** Fat energy is [Fat %] (threshold: < 30 %).
  [If ≥ 30 %: fat-source check — approved sources supply [share] % of total fat
  (threshold: ≥ 50 %) → PASS/FAIL. Breakdown: ...]

**Verdict: [Suitable / Not suitable] for diabetic individuals.**
[One line: if suitable, why all three pass; if not, which criterion/criteria failed and the number.]

- **Notes:** dietary screening from a representative recipe and IFCT-based estimates; restaurant
  portions and individual carbohydrate tolerance vary. Not personalised medical advice.
```

Round percentages and fibre to one decimal place; round the Atwater energy figures sensibly.
Show the fat-source breakdown whenever the fat-source check is triggered.

---

## Worked example 1 — Butter Chicken → Not suitable

`nutrition-calculator` returns, per 100 g: **Energy 171 kcal, Carbohydrate 3.3 g, Protein 8.8 g,
Fat 13.5 g, Dietary fibre 0.9 g** (representative restaurant recipe: boneless thigh in a
tomato-cashew-cream gravy finished with butter and ghee). Its ingredient ledger gives the recipe
**Fat (g) by ingredient**: chicken thigh 71.2, cashew paste 13.6, butter 32.4, fresh cream 20.0,
ghee 20.0, tomato 1.4, onion 0.2 — **total fat 158.8 g**.

**Energy contribution (Atwater):**

| Macronutrient | Energy (kcal) | % of total energy |
|---------------|--------------:|------------------:|
| Carbohydrate  | 3.3 × 4 = 13.2   | 7.8 %          |
| Protein       | 8.8 × 4 = 35.2   | 20.7 %         |
| Fat           | 13.5 × 9 = 121.5 | 71.5 %         |
| **Total**     | **169.9**        | **100 %**      |

**Criteria:**

- **Carbohydrate — PASS.** 7.8 % < 55 %.
- **Protein — PASS.** 20.7 % ≥ 20 % (no fibre fallback needed).
- **Fat — FAIL.** 71.5 % ≥ 30 %, so check the fat source. Approved-source fat = cashew (nuts)
  13.6 g only; chicken fat, butter, cream, and ghee are **not** approved. Share = 13.6 ÷ 158.8 =
  **8.6 %**, which is < 50 % → source check FAILS.

  | Fat source | Fat g | Approved? |
  |---|--:|---|
  | Chicken thigh fat | 71.2 | No (meat/poultry fat) |
  | Cashew paste | 13.6 | **Yes (nuts)** |
  | Butter | 32.4 | No |
  | Fresh cream | 20.0 | No |
  | Ghee | 20.0 | No |
  | Tomato / onion | 1.6 | No (negligible) |
  | **Total** | **158.8** | **8.6 % approved** |

**Verdict: Not suitable for diabetic individuals.** The fat criterion fails — fat supplies 71.5 %
of energy and only ~8.6 % of that fat comes from approved sources.

## Worked example 2 — Dal Makhani → Not suitable (shows the fibre fallback)

`nutrition-calculator` returns, per 100 g: **Energy 146 kcal, Carbohydrate 13.0 g, Protein 5.7 g,
Fat 7.4 g, Dietary fibre 5.3 g** (whole urad + rajmah, slow-simmered with tomato, butter and
cream). Recipe **Fat (g) by ingredient**: black gram 3.2, rajmah 0.9, onion 0.2, tomato 0.7,
butter 40.5, fresh cream 16.0, ghee 15.0 — **total fat 76.5 g**.

**Energy contribution (Atwater):**

| Macronutrient | Energy (kcal) | % of total energy |
|---------------|--------------:|------------------:|
| Carbohydrate  | 13.0 × 4 = 52.0  | 36.8 %         |
| Protein       | 5.7 × 4 = 22.8   | 16.1 %         |
| Fat           | 7.4 × 9 = 66.6   | 47.1 %         |
| **Total**     | **141.4**        | **100 %**      |

**Criteria:**

- **Carbohydrate — PASS.** 36.8 % < 55 %.
- **Protein — PASS (via fibre).** Protein is 16.1 % < 20 %, so fall back to fibre: 5.3 g/100 g ≥
  2 g → PASS.
- **Fat — FAIL.** 47.1 % ≥ 30 %, so check the fat source. Butter (40.5 g), cream (16.0 g), and
  ghee (15.0 g) dominate and are **not** approved; the pulses/onion/tomato fat is neither nuts nor
  seeds. Approved-source fat ≈ **0 %** of 76.5 g → source check FAILS.

**Verdict: Not suitable for diabetic individuals.** Carb passes and the fibre fallback rescues
protein, but fat fails — 47.1 % of energy from fat with essentially none of it from approved
sources.

## Worked example 3 — Bengali mustard-oil fish curry (Sorshe Maach) → Suitable

`nutrition-calculator` returns, per 100 g: **Energy 122 kcal, Carbohydrate 1.6 g, Protein 10.6 g,
Fat 6.6 g, Dietary fibre 0.4 g** (rohu simmered in a mustard-oil gravy). Recipe **Fat (g) by
ingredient**: rohu fish 7.0, mustard oil 45.0, yogurt 2.0, tomato 0.4, onion 0.2 — **total fat
54.6 g**.

**Energy contribution (Atwater):**

| Macronutrient | Energy (kcal) | % of total energy |
|---------------|--------------:|------------------:|
| Carbohydrate  | 1.6 × 4 = 6.4    | 5.9 %          |
| Protein       | 10.6 × 4 = 42.4  | 39.2 %         |
| Fat           | 6.6 × 9 = 59.4   | 54.9 %         |
| **Total**     | **108.2**        | **100 %**      |

**Criteria:**

- **Carbohydrate — PASS.** 5.9 % < 55 %.
- **Protein — PASS.** 39.2 % ≥ 20 % (no fibre fallback needed).
- **Fat — PASS (via source check).** 54.9 % ≥ 30 %, so check the fat source. Mustard oil (45.0 g)
  and the fish's own fat (7.0 g) are **both approved**; only yogurt (2.0 g) and vegetables (0.6 g)
  are not. Approved share = (45.0 + 7.0) ÷ 54.6 = **95.2 %** ≥ 50 % → source check PASSES.

  | Fat source | Fat g | Approved? |
  |---|--:|---|
  | Mustard oil | 45.0 | **Yes (mustard oil)** |
  | Rohu fish fat | 7.0 | **Yes (fish)** |
  | Yogurt | 2.0 | No (dairy fat) |
  | Tomato / onion | 0.6 | No (negligible) |
  | **Total** | **54.6** | **95.2 % approved** |

**Verdict: Suitable for diabetic individuals.** All three criteria pass — carbohydrate is low
(5.9 %), protein is high (39.2 %), and although fat supplies 54.9 % of energy, ~95 % of it comes
from approved sources (mustard oil and fish).

---

## Notes and edge cases

- **This skill depends on `nutrition-calculator`.** Never produce a verdict without first getting
  that skill's per-100 g values **and** its ingredient fat ledger. If the fat ledger is missing,
  the fat-source check can't be done — get it before finalising.
- **Threshold boundaries are exact.** Carb PASS is *strictly* below 55 %. Protein PASS is 20 % *or
  more*. Fibre fallback PASS is *at least* 2 g/100 g. Fat PASS is *strictly* below 30 %; the
  approved-fat source check passes at *50 % or more*. A value sitting exactly on a threshold follows
  the inclusive side stated here (e.g. protein exactly 20 % → PASS; fat exactly 30 % → go to the
  source check).
- **Denominator.** Percentages always use the Atwater total (Carb×4 + Protein×4 + Fat×9), so they
  sum to 100 %. Alcohol is not modelled.
- **Carbohydrate is used as reported** by `nutrition-calculator`; do not subtract fibre before the
  Atwater step.
- **All three criteria must pass** for "Suitable". One failure ⇒ "Not suitable"; always name the
  failing criterion and the number that breached its threshold.
- **Ambiguous fats.** If the dish's fat source can't be pinned down (e.g. unspecified "vegetable
  oil" vs mustard oil) and it would change the verdict, state the assumption and ask the user.
- **Scope.** This is macro/fibre-based screening, not a glycaemic-index measurement or clinical
  advice; note that individual tolerance and portion size vary.
