[SKILL.md](https://github.com/user-attachments/files/32282894/SKILL.md)
---
name: nutrition-calculator
description: >-
  Calculate per-100 g nutrition — Energy (kcal), Carbohydrate, Protein, Fat, and dietary
  fibre (total, plus its soluble/insoluble split and the dish's main fibre source) — for
  restaurant-style dishes from a chat description, an uploaded image, or a PDF of a menu/recipe.
  Use it whenever the user asks how many calories or how much protein/carb/fat/fibre a dish has,
  wants "macros per 100 g" for a cooked food, uploads or pastes a menu/recipe (any language) to
  analyse, or names a dish to estimate (e.g. butter chicken, biryani, paneer tikka). It picks an
  authentic restaurant recipe, looks ingredients up in the bundled Indian Food Composition Tables
  (IFCT), accounts for cooking yield/evaporation, and returns a per-100 g table of energy, the
  three macros, and fibre with its breakup and main source. Always report fibre — never stop
  at the four macros. Trigger even when the user doesn't say "IFCT", "fibre", or "per 100 g"
  but clearly wants a dish's nutrition.
---

# Nutrition Calculator (per 100 g, IFCT-based)

Estimate the nutrition of a restaurant-style dish **per 100 g of the finished, cooked
food**. The estimate is reproducible and evidence-based: every ingredient value comes
from the bundled Indian Food Composition Tables (IFCT), and the cooked weight is
reasoned from the recipe rather than guessed. This is an estimate of a representative
preparation, not a claim about one restaurant's exact recipe or portion.

## Inputs this skill handles

The dish may arrive as a plain chat description ("nutrition for butter chicken"), an
**image** of a menu or recipe, or a **PDF**. Read every supplied attachment with the
appropriate tool (image inspection, PDF text extraction). Menus/recipes in another
language must be **translated** to English or the common Indian food name before you can
match ingredients to IFCT (see Step 3). If the user names several dishes, run the
workflow for each.

## Data sources (in priority order)

1. **`references/ifct-index.md`** — a parsed, per-100 g table of all 528 IFCT foods
   (Energy already converted to kcal, Carbohydrate, Protein, Fat, and **dietary fibre —
   Total / Insoluble / Soluble**). This is your fast lookup. It has an *Averaged variety
   groups* section and an *All foods* table. Grep it.
2. **`references/ifct-proximate-raw.txt`** — the verbatim IFCT proximate table text. Use it
   to confirm anything ambiguous or to read a value the index abbreviates.
3. **`assets/IFCT_edited.pdf`** — the bundled source of truth (a trimmed copy of the IFCT
   2017 book, keeping the proximate-composition tables). If a value isn't in the index/raw
   file, read it here (e.g. with `pypdf`). Energy in the PDF is in **kJ** — convert to kcal.
4. **`references/oil-ghee-constants.md`** — fixed values for cooking oil and ghee.
5. **`references/ingredient-fallback-notes.md`** — what to do when an ingredient is not
   in IFCT (use Indian packaged-food labels), with typical values and unit conversions.

Use IFCT values **directly as printed** — do not "correct" or recompute them. The only
transformation allowed on a looked-up value is the kJ→kcal energy conversion below.

**Column order — read carefully.** Every nutrient table in this skill (the index, the
fallback tables, the worked-example ledgers, and the output) lists nutrients in the **same
fixed order: Energy (kcal), Carbohydrate, Protein, Fat**, then **dietary fibre: Total,
Insoluble, Soluble**. When you see a compact source note like `N002 (137/0/20/5.8)`, that
is `kcal / Carb / Protein / Fat`; a fibre note like `fib 1.7/1.4/0.3` is `Total / Insoluble /
Soluble`. Keeping one order everywhere is what stops a fat value being read into a protein
column, or insoluble fibre into soluble.

---

## The procedure

Follow these six steps in order. Steps 2 (cooked yield) and 3 (nutrition lookup) are
where estimates go wrong, so they carry the most detail.

### Step 1 — Choose an authentic, restaurant-style recipe

Identify the recipe most representative of how the dish is actually cooked in a
restaurant. Prefer a quantified recipe (ingredient weights/volumes and, ideally, a stated
number of servings). Match the real thing: the right cuisine, technique, sauce, garnish,
and the fats a restaurant actually uses (butter, cream, cashew paste, ghee). **Do not**
pick a "healthy", low-fat, or slimmed-down home version unless the user asks for that —
restaurant food is richer, and choosing a light recipe understates the result. Note what
recipe you used so the estimate is auditable.

### Step 2 — Recipe yield: the cooked weight in grams

You need the **total weight of the finished, cooked dish**, because per-100 g means "per
100 g of what's served". This is **not** the sum of raw ingredient weights. Cooking moves
water (and sometimes oil) in and out of the food:

- **Simmered/reduced dishes** (curries, gravies, sauces, most restaurant mains): water
  evaporates, so the cooked weight is **less** than the raw sum. Added water for a gravy
  largely boils off. A typical simmered curry loses roughly **10–25 %** of its in-pan mass
  to evaporation — reason from how long and how hard it's reduced.
- **Boiled pulses, rice, pasta**: the dry grain/pulse **absorbs** water and swells. Dry
  pulses roughly 2.5–3× their weight when cooked; rice roughly 2.5–3×; pasta roughly 2.2×.
  Here the cooked weight is **more** than the raw dry weight. Count only the water the food
  **retains**, not all the water it was boiled in (the rest is drained or evaporates).
- **Deep-fried foods**: some frying oil is absorbed and retained (adds weight); water is
  driven off (loses weight). As an anchor, a battered/breaded coating (pakora, tempura,
  cutlet, bhaji) typically absorbs roughly **10–20 % of its own weight in oil**, while the
  food loses roughly **10–20 %** of its water — so the net weight change is often small, but
  fat and energy jump from the retained oil (value that oil with the oil/ghee constant).
  Scale the absorbed oil to how heavy/thick the coating is.
- **Baked doughs / breads** (pav, bun, naan, kulcha, roti/chapati, pizza base, biscuits,
  cake): the dough loses moisture in the oven/tandoor/griddle, so the baked item weighs
  **less** than the dough. Bread, pav, and buns lose roughly **10–18 %** of dough weight
  (thinner/crustier bakes lose more); soft naan/kulcha ~10–15 %. Build the ledger from the
  dry flour + water + fat as raw ingredients (Step 3's raw-state rule), and take the baked
  weight as dough minus this loss. Add any griddling/finishing butter separately. **Steamed**
  batters (idli, dhokla, momo) instead **retain** their water — treat them like the
  absorption case above.
- **Assembled / composite items** (rolls, wraps, sandwiches, burgers, thalis, platters,
  rice bowls): do **not** apply one factor to the whole thing. Work out the cooked weight of
  each component **separately**, then add them up. Components that are cooked lose water —
  grilled/tandoori/roasted meat roughly **20–30 %**, a griddled or toasted flatbread/roti/bun
  roughly **10–20 %**, cooked rice per the absorption rule above — while **cold add-ins**
  (sauces, mayo, chutney, raw salad, pickle, cheese slices) keep their raw weight. The
  finished item's yield is the sum of the cooked-component weights plus the cold ones.

Write down the assumption you used ("simmered ~20 min, ~18 % evaporative loss → cooked
yield ≈ N g", or "250 g dry pulses absorbed water to ≈ 640 g cooked"). If a recipe states
its finished weight or a serving count with portion size, use that instead.

**Key point:** water carries no calories or macros, so evaporation/absorption does **not**
change the recipe's nutrient totals from Step 3 — it only changes the yield you divide by.
More reduction ⇒ smaller yield ⇒ **denser** per-100 g numbers. The same cooking factors work
in reverse: if a recipe states an ingredient in *cooked* terms, use them to recover its raw
mass for the Step 3 lookup (see Step 3, point 3). Note the dish's mass **includes the spices,
herbs and aromatics that Step 3 excludes from the nutrition** — count their weight here in the
yield even though they add no energy or macros.

### Step 3 — Recipe nutrition: total Energy, Carb, Protein, Fat

Build an ingredient ledger. **Spices, herbs, and the aromatics listed below are excluded from
the nutrition but kept in the weight** (see the boundary below). For each *counted* edible
ingredient, in grams:

**Excluded from energy/macros — but their weight is retained.** For spices, herbs, and the
aromatics listed below: **do not look them up or count their Energy, Carbohydrate, Protein, or
Fat** — treat each as contributing **zero** to the nutrient totals (their nutritional
contribution is small and several are unreliable to find in a composition table). **But keep
their weight**: include their grams in the recipe's raw/cooked mass so they still count toward
the Step 2 yield (the denominator). The effect is that they bulk out the finished dish without
adding calories, which slightly **lowers** the per-100 g figures. The items to treat this way:

- All whole/ground **spices** and spice blends — cumin/jeera, coriander seed, black pepper,
  turmeric/haldi, chilli powder, cardamom, clove, cinnamon, bay leaf, mustard/rai,
  fenugreek/methi seed, asafoetida/hing, nutmeg, star anise, garam masala, and
  pav-bhaji / sambar / rasam / chaat masala, etc.
- All fresh or dried **herbs / leaves** — coriander/cilantro (leaves *and* seeds), mint/pudina,
  curry leaves, kasuri methi, basil, parsley, dill.
- **Ginger, garlic** (and ginger-garlic paste), **tamarind/imli**, and **any raw/fresh chilli**
  (green or red).
- **Salt** and other pure seasonings (they carry no energy/macros regardless; their small
  weight can be counted too).

**Counted normally** (both nutrition *and* weight): every substantive ingredient — among the
aromatics that means **onion and tomato** (and their purées/pastes), plus everything else
(meats, dairy, pulses, vegetables, nuts, flours, oils, sauces). If unsure whether something is
a seasoning-scale flavouring (exclude from nutrition, keep weight) or a base ingredient (count
fully), judge by quantity. In the ledger, list the excluded items as a single weight-only row
with zero in every nutrient column, so their grams flow into the yield.

1. **Translate** foreign/regional names to match IFCT (e.g. *baingan*→brinjal/eggplant,
   *murgh*→chicken, *jeera*→cumin, *besan*→Bengal gram flour). See the fallback file for a
   translation starter list.
2. **Look up per-100 g values** in `references/ifct-index.md` (columns: Energy kcal,
   Carbohydrate, Protein, Fat, then **dietary fibre — Total, Insoluble, Soluble** — energy
   already converted). Pick the right form (raw vs roasted, whole vs dal, fresh vs dried).
   Exclude inedible parts (bones, shells, peels) and any oil explicitly discarded after
   cooking; include fats retained in the dish.
   - **Fibre sources.** Animal-flesh foods, egg, milk, paneer/khoa, sugar, and **oil/ghee**
     have **zero** fibre. Fibre comes from plant foods — pulses, whole grains, vegetables,
     nuts, fruit. For a **fallback** ingredient not in IFCT, read **total** dietary fibre off
     the Indian label; the soluble/insoluble split is usually not printed, so put the total
     under *Insoluble* (the larger share for most foods) unless you have a better figure, and
     say so.
3. **Raw vs cooked — match the quantity's state to the value.** IFCT per-100 g values are for
   the food **raw** (unless the entry name says "boiled"/"roasted"). A nutrient total depends
   only on each ingredient's **raw/dry mass**, because cooking just moves water and water has
   no calories. Read each recipe line and see which state its quantity is written in:
   - **Raw/dry quantity** ("60 g rice", "200 g chicken", "1 cup dry urad") → apply the raw
     IFCT per-100 g values directly to that weight.
   - **Cooked/hydrated quantity** ("2 cups *cooked* rice", "150 g *boiled* dal", "1 cup
     *blanched* spinach", "1 cup shredded *cooked* chicken") → do **not** put raw calorie
     density on a cooked weight; a hydrated food overcounts by the water it soaked up, a
     shrunken one undercounts. First convert back to the **raw-equivalent** weight using the
     same cooking factors as Step 2 — rice/pasta/pulses cooked ≈ 2.5–3× raw (so raw ≈ cooked
     ÷ ~2.7); leafy greens cooked ≈ 0.4× raw (raw ≈ cooked ÷ 0.4); grilled/roasted meat
     cooked ≈ 0.75× raw (raw ≈ cooked ÷ 0.75) — then apply the raw per-100 g values.

   Keep this distinct from Step 2: **nutrition is computed from the raw/dry mass; the cooked
   mass is only the yield you divide by in Step 4.** (E.g. 60 g raw rice → nutrition from 60 g;
   its ~158 g cooked weight feeds only the yield.)
4. **Duplicates → average.** IFCT often lists the same food as several regional/variety
   rows (e.g. Brinjal-1…Brinjal-12, onion big/small, tomato local/hybrid, several potato
   rows). When your ingredient matches such a group rather than one specific variety,
   **average their per-100 g values**. The index's *Averaged variety groups* section
   pre-computes the numbered-variant ones; IFCT's own "all varieties" rows may be used
   directly.
5. **Oil and ghee → fixed constants.** For **any** cooking oil or ghee, always use exactly
   **Energy 1000 kcal, Carb 0 g, Protein 0 g, Fat 100 g per 100 g/ml** — never look these
   up elsewhere. (Butter, cream, and other dairy fats are *not* oil/ghee — look them up.)
   See `references/oil-ghee-constants.md`.
6. **Not in IFCT → fallback.** If an ingredient genuinely isn't in the PDF (even after
   translating), use typical values from an Indian packaged-food label, and document the
   source. See `references/ingredient-fallback-notes.md`.
7. **Energy unit.** IFCT prints Energy in **kJ**; the index already shows **kcal**. If you
   read a raw kJ value from the PDF, convert: **kcal = kJ ÷ 4.184**.

Each ingredient's contribution = `per-100 g value × grams ÷ 100`. Sum contributions across
all ingredients to get the recipe's total Energy, Carbohydrate, Protein, Fat, and dietary
fibre (Total, Insoluble, Soluble). Also note **which ingredient contributes the most total
fibre** — that's the dish's *main fibre source*, reported in Step 5. (Soluble + Insoluble
should roughly equal Total; small IFCT rounding differences are fine.)

### Step 4 — Nutrition per 100 g

For each nutrient — Energy, Carb, Protein, Fat, and each of the three fibre values:

```
per_100g = (recipe total for that nutrient / cooked recipe yield in grams) × 100
```

### Step 5 — Output the table

The **Dietary fibre** column and the two fibre lines (the soluble/insoluble split and the main
fibre source) are a **required** part of every result — always fill them in, even when fibre is
low or zero (e.g. a meat-and-cream dish comes out near 0 g; still report it). Do not stop at the
four macros.

Present the result using this structure:

```markdown
## [Dish name] — nutrition per 100 g

| Energy (kcal) | Carbohydrate (g) | Protein (g) | Fat (g) | Dietary fibre (g) |
|--------------:|-----------------:|------------:|--------:|------------------:|
|          [E]  |            [C]    |       [P]   |   [F]   |        [FibTotal] |

- **Dietary fibre:** [Total] g total — **[Insoluble] g insoluble, [Soluble] g soluble**.
- **Main fibre source:** [ingredient] (~[x] g of the [Total] g total fibre).
- **Cooked yield used:** [N] g — [one line on the evaporation/absorption basis]
- **Recipe basis:** [what recipe / source]
- **Notes:** estimate from a representative recipe and IFCT values; restaurant portions
  and preparation vary.
```

Round Energy to a whole number and Carbohydrate, Protein, Fat, and each fibre value to one
decimal place. Show the ingredient ledger (ingredient, grams, IFCT entry, contribution)
when the user wants the working or asks how you got there.

### Step 6 — Ask before finalizing on high-impact ambiguities

Before you commit to numbers, ask the user any clarifying question where a reasonable
person could go two ways **and it materially changes the result**. Don't silently guess on
these. Typical ones:

- **Cut/protein form** — chicken breast vs thigh vs leg (with/without skin); paneer vs tofu.
- **Fat used** — ghee vs oil vs butter; how much; cream vs yogurt vs cashew for richness.
- **Regional variant** — e.g. a "korma" or "biryani" style that changes ingredients a lot.
- **Portion/units** — if the recipe uses vague measures ("a cup of gravy") that swing the yield.

Ask a small number of pointed questions (not an interrogation), then finish the
calculation. If the user says "just assume something reasonable", state your assumptions
and proceed.

---

## Worked example 1 — Butter Chicken

A representative restaurant recipe (boneless thigh, tomato-cashew-cream gravy finished with
butter). Ingredient ledger — each contribution = per-100 g × g ÷ 100:

Source notes are `kcal/C/P/F · fib T/I/S`, matching the ledger's Energy, Carb, Protein, Fat,
then Fibre-Total/Insoluble/Soluble columns.

| Ingredient | Grams | IFCT / source (kcal/C/P/F · fib T/I/S) | Energy kcal | Carb g | Protein g | Fat g | Fib-T g | Fib-I g | Fib-S g |
|---|--:|---|--:|--:|--:|--:|--:|--:|--:|
| Chicken, thigh, skinless | 500 | N002 (137/0/20/5.8 · 0/0/0) — user-overridden | 685 | 0 | 100.0 | 29.0 | 0 | 0 | 0 |
| Tomato, ripe (avg) | 400 | IFCT D075/D076 avg (19.5/2.96/0.83/0.36 · 1.68/1.35/0.32) | 78 | 11.8 | 3.3 | 1.4 | 6.7 | 5.4 | 1.3 |
| Onion (avg big+small) | 120 | IFCT G017/G018 avg (52.5/10.57/1.66/0.20 · 1.81/1.31/0.49) | 63 | 12.7 | 2.0 | 0.2 | 2.2 | 1.6 | 0.6 |
| Cashewnut (paste) | 30 | IFCT H005 (583/25.46/18.78/45.2 · 3.86/2.23/1.63) | 175 | 7.6 | 5.6 | 13.6 | 1.2 | 0.7 | 0.5 |
| Butter | 40 | fallback (717/0/0.5/81 · 0) | 287 | 0 | 0.2 | 32.4 | 0 | 0 | 0 |
| Fresh cream ~25 % | 100 | fallback (210/6.5/2.2/20 · 0) | 210 | 6.5 | 2.2 | 20.0 | 0 | 0 | 0 |
| **Ghee** | 20 | **oil/ghee constant (1000/0/0/100 · 0)** | 200 | 0 | 0.0 | 20.0 | 0 | 0 | 0 |
| Salt | 8 | (0…) | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Water (for gravy) | 150 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Spices, ginger, garlic, herbs — *excluded from nutrition, weight only* | 46 | 0 (weight only) | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| **Recipe total** | **1414 raw** | | **1698** | **38.7** | **113.3** | **116.6** | **10.0** | **7.7** | **2.3** |

*Per Step 3, the 46 g of spices + ginger + garlic (red chilli, turmeric, garam masala, kasuri
methi, ginger, garlic) are **excluded from the energy/macro totals but their weight is kept** —
so they enlarge the yield without adding calories, which pulls the per-100 g figures down a
little.*

**Yield (Step 2):** the gravy is simmered and reduced; essentially all 150 g added water
boils off and the tomato/onion/cream/aromatic moisture reduces — an evaporative loss of ≈ 236 g
(~17 % of the 1414 g in-pan mass, which now includes the 46 g of weight-only spices/aromatics).
**Cooked yield ≈ 1178 g.**

**Per 100 g (Step 4):** total ÷ 1178 × 100 →

| Energy (kcal) | Carbohydrate (g) | Protein (g) | Fat (g) | Dietary fibre (g) |
|--------------:|-----------------:|------------:|--------:|------------------:|
|          144  |             3.3   |        9.6  |    9.9  |               0.9 |

- **Dietary fibre:** 0.9 g total — **0.7 g insoluble, 0.2 g soluble**.
- **Main fibre source:** tomato (~6.7 g of the ~10.0 g total recipe fibre; the meat, dairy and
  fats contribute none).
- **Cooked yield used:** 1178 g — simmered gravy, ~17 % loss; includes weight-only spices/aromatics.
- **Notes:** thigh (not breast) and butter+cream+ghee reflect restaurant richness; ask the
  user if they want breast or a leaner gravy.

## Worked example 2 — Dal Makhani

Whole black gram (urad) + red kidney beans, slow-simmered with tomato, butter and cream.
The teaching contrast with Example 1: here the dry pulses **absorb** water (raising yield),
while the long simmer still evaporates some. Water carries no nutrients, so it changes only
the yield, not the totals.

Source notes are `kcal/C/P/F · fib T/I/S`, matching the ledger's Energy, Carb, Protein, Fat,
then Fibre-Total/Insoluble/Soluble columns.

| Ingredient | Grams | IFCT / source (kcal/C/P/F · fib T/I/S) | Energy kcal | Carb g | Protein g | Fat g | Fib-T g | Fib-I g | Fib-S g |
|---|--:|---|--:|--:|--:|--:|--:|--:|--:|
| Black gram, whole (urad), dry | 200 | IFCT B004 (291/43.99/21.97/1.58 · 20.41/15.47/4.94) | 582 | 88.0 | 43.9 | 3.2 | 40.8 | 30.9 | 9.9 |
| Rajmah, red (kidney bean), dry | 50 | IFCT B020 (299/48.61/19.91/1.77 · 16.57/13.86/2.7) | 150 | 24.3 | 10.0 | 0.9 | 8.3 | 6.9 | 1.4 |
| Onion (avg big+small) | 100 | IFCT G017/G018 avg (52.5/10.57/1.66/0.20 · 1.81/1.31/0.49) | 53 | 10.6 | 1.7 | 0.2 | 1.8 | 1.3 | 0.5 |
| Tomato, ripe (avg) | 200 | IFCT D075/D076 avg (19.5/2.96/0.83/0.36 · 1.68/1.35/0.32) | 39 | 5.9 | 1.7 | 0.7 | 3.4 | 2.7 | 0.6 |
| Butter | 50 | fallback (717/0/0.5/81 · 0) | 359 | 0 | 0.3 | 40.5 | 0 | 0 | 0 |
| Fresh cream ~25 % | 80 | fallback (210/6.5/2.2/20 · 0) | 168 | 5.2 | 1.8 | 16.0 | 0 | 0 | 0 |
| **Ghee** | 15 | **oil/ghee constant (1000/0/0/100 · 0)** | 150 | 0 | 0.0 | 15.0 | 0 | 0 | 0 |
| Salt | 6 | (0…) | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| Spices, ginger, garlic — *excluded from nutrition, weight only* | 39 | 0 (weight only) | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| **Recipe total** | | | **1500** | **134.0** | **59.2** | **76.5** | **54.3** | **41.9** | **12.4** |

*Per Step 3, the 39 g of red chilli, garam masala, ginger and garlic are **excluded from the
energy/macro totals but their weight is retained** in the yield.*

**Yield (Step 2):** the 250 g of dry pulses absorb cooking water and swell to ≈ 640 g; the
other ingredients (~490 g, **including the 39 g of weight-only spices/aromatics**) lose some
moisture over the long simmer. Net **cooked yield ≈ 1030 g** (the litre-plus of cooking water
it was boiled in is mostly evaporated/absorbed — do **not** add all of it to the yield).

**Per 100 g (Step 4):** total ÷ 1030 × 100 →

| Energy (kcal) | Carbohydrate (g) | Protein (g) | Fat (g) | Dietary fibre (g) |
|--------------:|-----------------:|------------:|--------:|------------------:|
|          146  |            13.0   |        5.7  |    7.4  |               5.3 |

- **Dietary fibre:** 5.3 g total — **4.1 g insoluble, 1.2 g soluble**.
- **Main fibre source:** whole urad / black gram (~40.8 g of the ~54.3 g total recipe fibre,
  ~75 %; rajmah adds most of the rest).
- **Cooked yield used:** 1030 g — dry pulses hydrated + long simmer.
- **Notes:** restaurant dal makhani is butter/cream-heavy; a homestyle version with less
  fat would come out lower — worth confirming with the user. The pulses make this a
  genuinely high-fibre dish, unlike the meat-based Butter Chicken.

---

## Rebuilding the index (maintenance)

`references/ifct-index.md` and `references/ifct-proximate-raw.txt` are generated from
`assets/IFCT_edited.pdf` by `scripts/build_ifct_index.py` (needs `pypdf`; it auto-detects the
proximate pages, so it also works on the full IFCT.pdf). Re-run it only if the bundled PDF is
replaced:

```bash
python3 scripts/build_ifct_index.py
```
