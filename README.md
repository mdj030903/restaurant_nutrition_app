# restaurant_nutrition_app
**What it does**
Collects users data- height, weight, health goal, clinical conditions, etc to roughly estimate the nutrition requirement (not shared with user)
Calculates the nutrition values of dishes on a menu uploaded by the user. Nutrition is calculated per 100g and per portion using data from the Indian Food and Composition Tables.
Provides a verdict on whether the dish is suitable for them based on the estimated nutrition requirement. If the user is diabetic, it runs another test to verify if the dishes are suitable for them based on pre-determined criteria (carbohydrate, protein, fat and fibre ratio)
Gives suggestions about what can be added or eliminated, how much of the portion can be eaten to help the client stick to their goals.

**Why am i building it?**
To help people stick to their health goals without feeling like they've sabotaged their progress. Having an app helping you make the decision leaves the guesswork out and helps people identify what would work best for them, no matter where they are. The diabetes- suitability feature is especially to enable people to enjoy social gatherings and outings without feeling isolated and to help them keep their blood sugar in check while still enjoying what's available.

**Where is it now?**
Currently in the phase of building a database with at least 5 restaurant menus. This will help me pilot the system with a small sample. The aim is to get an accredited and well-known nutritionist to give their seal of approval on the app and eventually work directly with restaurants to get certified nutrition data.

**How am i building it?**
Using Claude Code to map out the steps and create a Supabase database. Two skills are already created on using Claude code to calculate the nutrition values and determine the appropriateness for a diabetic.
