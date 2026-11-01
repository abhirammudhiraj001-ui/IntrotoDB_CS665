# Project Check-in 2 — CS-665

## Normalization & Schema Evolution: Recipe Finder Mobile Application

**Abhiram Golkonda** · B855Q929 · 

---

## 1. Schema Evolution & Core Architectural Changes

### 1.1 Why the Check-in 1 Schema Had to Grow

In Check-in 1, I worked with a basic two-table setup: **Recipes** and **Ingredients**. That simple structure was fine for basic lookups like "Which recipes use chicken?", but as I started planning out the app's actual features, I realized it falls short. The real goal of the app is to answer: **"What can I cook with what I already have in my kitchen right now?"**

To support that workflow, the database design had to evolve for a few main reasons:

- **A dedicated pantry:** The app needs to keep track of what ingredients a user currently owns, completely separate from any specific recipe.
- **Ingredients as independent entities:** An item like "Onion" should only exist once in the database catalog, regardless of how many recipes use it.
- **Meal categories:** Users need a way to filter recipes easily by categories like Breakfast, Lunch, or Dinner.
- **Separating amounts and units:** Storing text like `500 g` or `2 cloves` as a single field makes it impossible to scale quantities mathematically, run inventory comparisons, or validate user inputs properly.

### 1.2 Final Table Structures & Relationships

| Entity | Purpose | Primary Key |
|---|---|---|
| Categories | Stores meal categories (Breakfast, Lunch, Dinner) | `category_id` |
| Recipes | Master record for saved recipes and instructions | `recipe_id` |
| Ingredients | Single canonical list of distinct ingredients | `ingredient_id` |
| Units | Standard measurement units (g, tbsp, clove, whole, head) | `unit_id` |
| Recipe_Ingredients | Junction table matching ingredients, amounts, and units to recipes | (`recipe_id`, `ingredient_id`) |
| Pantry | Tracks items the user currently has on hand | `ingredient_id` |

| Relationship | Cardinality | How It Is Implemented |
|---|---|---|
| Category → Recipe | One-to-Many | `Recipes.category_id` (Foreign Key) |
| Recipe ↔ Ingredient | Many-to-Many | Junction table `Recipe_Ingredients` |
| Unit → Recipe_Ingredients | One-to-Many | `Recipe_Ingredients.unit_id` (Foreign Key) |
| Ingredient → Pantry | One-to-Zero-or-One | `Pantry.ingredient_id` is both PK and FK |

The biggest shift from Check-in 1 is how recipes connect to ingredients. Previously, I modeled it as a one-to-many relationship (as if an ingredient belonged to just one recipe). In reality, common items like Onion or Tomato are used in many different dishes, so it is a true many-to-many relationship that requires a dedicated junction table.

---

## 2. Step-by-Step Normalization Demonstration

To show that breaking down the tables doesn't lose any facts, I am keeping the exact same **4 recipes and 11 total ingredient items** consistent through every step of normalization.

### 2.0 Unnormalized Form (UNF)

In UNF, all data for a recipe is stored in a single large row, keeping ingredients as a repeating list inside a single column.

| recipe_id | recipe_name | category_id | category_name | instructions | ingredients (name, quantity) |
|---|---|---|---|---|---|
| 1 | Chicken Curry | 1 | Dinner | Saute onion, add chicken and curry powder, simmer. | Chicken 500 g; Onion 1; Curry Powder 2 tbsp |
| 2 | Tomato Pasta | 1 | Dinner | Boil pasta, cook tomato and garlic, combine. | Pasta 200 g; Tomato 3; Garlic 2 cloves |
| 3 | Veggie Omelette | 2 | Breakfast | Beat eggs, add onion and tomato, cook in a pan. | Eggs 3; Onion 1; Tomato 1 |
| 4 | Chicken Salad | 3 | Lunch | Chop cooked chicken, mix with lettuce. | Chicken 200 g; Lettuce 1 head |

### 2.1 First Normal Form (1NF)

**Rule:** Ensure every cell holds a single atomic value, remove repeating groups, and set a primary key.

**Changes made:** I flattened the repeating ingredient lists into individual rows and broke down quantities into separate numerical amounts and unit types. The primary key becomes the composite pair (`recipe_id`, `ingredient_name`).

| recipe_id | ingredient_name | amount | unit | recipe_name | category_id | category_name |
|---|---|---|---|---|---|---|
| 1 | Chicken | 500 | g | Chicken Curry | 1 | Dinner |
| 1 | Onion | 1 | whole | Chicken Curry | 1 | Dinner |
| 1 | Curry Powder | 2 | tbsp | Chicken Curry | 1 | Dinner |
| 2 | Pasta | 200 | g | Tomato Pasta | 1 | Dinner |
| 2 | Tomato | 3 | whole | Tomato Pasta | 1 | Dinner |
| 2 | Garlic | 2 | clove | Tomato Pasta | 1 | Dinner |
| 3 | Eggs | 3 | whole | Veggie Omelette | 2 | Breakfast |
| 3 | Onion | 1 | whole | Veggie Omelette | 2 | Breakfast |
| 3 | Tomato | 1 | whole | Veggie Omelette | 2 | Breakfast |
| 4 | Chicken | 200 | g | Chicken Salad | 3 | Lunch |
| 4 | Lettuce | 1 | head | Chicken Salad | 3 | Lunch |

**Verification:** We have 11 rows total, matching our original dataset. However, general recipe information is repeated across every ingredient row, showing clear partial key dependencies.

### 2.2 Second Normal Form (2NF)

**Rule:** Must be in 1NF, and every non-key column must depend on the complete composite key (no partial dependencies).

**Functional Dependencies in 1NF:**

- `recipe_id → recipe_name, category_id, category_name, instructions` (Partial dependency: depends only on `recipe_id`, not the ingredient)
- `(recipe_id, ingredient_name) → amount, unit` (Full functional dependency)

**Resolution:** I split the table into two so that general recipe attributes are separated from specific ingredient amounts.

**Table: Recipes_2NF** (Primary Key: `recipe_id`)

| recipe_id | recipe_name | category_id | category_name | instructions |
|---|---|---|---|---|
| 1 | Chicken Curry | 1 | Dinner | Saute onion, add chicken and curry powder, simmer. |
| 2 | Tomato Pasta | 1 | Dinner | Boil pasta, cook tomato and garlic, combine. |
| 3 | Veggie Omelette | 2 | Breakfast | Beat eggs, add onion and tomato, cook in a pan. |
| 4 | Chicken Salad | 3 | Lunch | Chop cooked chicken, mix with lettuce. |

**Table: RecipeIngredients_2NF** (Primary Key: `recipe_id`, `ingredient_name`)

| recipe_id | ingredient_name | amount | unit |
|---|---|---|---|
| 1 | Chicken | 500 | g |
| 1 | Onion | 1 | whole |
| 1 | Curry Powder | 2 | tbsp |
| 2 | Pasta | 200 | g |
| 2 | Tomato | 3 | whole |
| 2 | Garlic | 2 | clove |
| 3 | Eggs | 3 | whole |
| 3 | Onion | 1 | whole |
| 3 | Tomato | 1 | whole |
| 4 | Chicken | 200 | g |
| 4 | Lettuce | 1 | head |

### 2.3 Third Normal Form (3NF)

**Rule:** Must be in 2NF, and no non-key attribute can depend on another non-key attribute (eliminate transitive dependencies).

**Transitive Dependency Found:** In `Recipes_2NF`, we have `recipe_id → category_id → category_name`. The category name depends on the category ID, not directly on the recipe itself.

**Further Decomposition:** I moved categories into their own table. Additionally, I extracted Ingredients and Units into lookup tables so that misspelled text entries won't break search or pantry matching.

**Table: Categories** (PK: `category_id`)

| category_id | category_name |
|---|---|
| 1 | Dinner |
| 2 | Breakfast |
| 3 | Lunch |

**Table: Units** (PK: `unit_id`)

| unit_id | unit_name |
|---|---|
| 1 | g |
| 2 | tbsp |
| 3 | clove |
| 4 | head |
| 5 | whole |

**Table: Recipes** (PK: `recipe_id`, FK: `category_id`)

| recipe_id | recipe_name | category_id | instructions |
|---|---|---|---|
| 1 | Chicken Curry | 1 | Saute onion, add chicken and curry powder, simmer. |
| 2 | Tomato Pasta | 1 | Boil pasta, cook tomato and garlic, combine. |
| 3 | Veggie Omelette | 2 | Beat eggs, add onion and tomato, cook in a pan. |
| 4 | Chicken Salad | 3 | Chop cooked chicken, mix with lettuce. |

**Table: Ingredients** (PK: `ingredient_id`)

| ingredient_id | ingredient_name |
|---|---|
| 1 | Chicken |
| 2 | Onion |
| 3 | Curry Powder |
| 4 | Pasta |
| 5 | Tomato |
| 6 | Garlic |
| 7 | Eggs |
| 8 | Lettuce |

**Table: Recipe_Ingredients** (PK: `recipe_id`, `ingredient_id`)

| recipe_id | ingredient_id | amount | unit_id |
|---|---|---|---|
| 1 | 1 | 500 | 1 |
| 1 | 2 | 1 | 5 |
| 1 | 3 | 2 | 2 |
| 2 | 4 | 200 | 1 |
| 2 | 5 | 3 | 5 |
| 2 | 6 | 2 | 3 |
| 3 | 7 | 3 | 5 |
| 3 | 2 | 1 | 5 |
| 3 | 5 | 1 | 5 |
| 4 | 1 | 200 | 1 |
| 4 | 8 | 1 | 4 |

**Table: Pantry** (PK/FK: `ingredient_id`) — Sample data for user on hand

| ingredient_id | date_added | Corresponding Ingredient |
|---|---|---|
| 1 | 2026-10-20 | Chicken |
| 2 | 2026-10-20 | Onion |
| 5 | 2026-10-22 | Tomato |
| 6 | 2026-10-22 | Garlic |
| 7 | 2026-10-25 | Eggs |

### 2.4 Proof of Lossless-Join Decomposition

Joining all 3NF tables together accurately reconstructs the full 11-row 1NF dataset without missing information or creating false data:

```sql
SELECT r.recipe_id, i.ingredient_name, ri.amount, u.unit_name,
       r.recipe_name, c.category_id, c.category_name
FROM Recipes r
JOIN Categories c ON c.category_id = r.category_id
JOIN Recipe_Ingredients ri ON ri.recipe_id = r.recipe_id
JOIN Ingredients i ON i.ingredient_id = ri.ingredient_id
JOIN Units u ON u.unit_id = ri.unit_id
ORDER BY r.recipe_id, i.ingredient_id;
```

### 2.5 Final 3NF DDL Schema (SQL Statements)

```sql
CREATE TABLE Categories (
    category_id INT PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE Units (
    unit_id INT PRIMARY KEY,
    unit_name VARCHAR(30) NOT NULL UNIQUE
);

CREATE TABLE Recipes (
    recipe_id INT PRIMARY KEY,
    recipe_name VARCHAR(255) NOT NULL,
    category_id INT NOT NULL,
    instructions TEXT,
    FOREIGN KEY (category_id) REFERENCES Categories(category_id)
);

CREATE TABLE Ingredients (
    ingredient_id INT PRIMARY KEY,
    ingredient_name VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE Recipe_Ingredients (
    recipe_id INT NOT NULL,
    ingredient_id INT NOT NULL,
    amount DECIMAL(8,2) NOT NULL,
    unit_id INT NOT NULL,
    PRIMARY KEY (recipe_id, ingredient_id),
    FOREIGN KEY (recipe_id) REFERENCES Recipes(recipe_id) ON DELETE CASCADE,
    FOREIGN KEY (ingredient_id) REFERENCES Ingredients(ingredient_id),
    FOREIGN KEY (unit_id) REFERENCES Units(unit_id)
);

CREATE TABLE Pantry (
    ingredient_id INT PRIMARY KEY,
    date_added DATE,
    FOREIGN KEY (ingredient_id) REFERENCES Ingredients(ingredient_id)
);
```

### 2.6 Core Application Queries

**Query 1: Ingredient Search** (e.g., "Find all recipes using Chicken")

```sql
SELECT r.recipe_name
FROM Recipes r
JOIN Recipe_Ingredients ri ON ri.recipe_id = r.recipe_id
JOIN Ingredients i ON i.ingredient_id = ri.ingredient_id
WHERE i.ingredient_name = 'Chicken';
-- Output: Chicken Curry, Chicken Salad
```

**Query 2: Pantry Recipe Matcher** ("What can I cook with what I have?")

```sql
SELECT r.recipe_name,
       COUNT(*) AS total_ingredients,
       SUM(CASE WHEN p.ingredient_id IS NULL THEN 1 ELSE 0 END) AS missing_ingredients
FROM Recipes r
JOIN Recipe_Ingredients ri ON ri.recipe_id = r.recipe_id
LEFT JOIN Pantry p ON p.ingredient_id = ri.ingredient_id
GROUP BY r.recipe_id, r.recipe_name
ORDER BY missing_ingredients ASC;
```

| recipe_name | total_ingredients | missing_ingredients | App Result |
|---|---|---|---|
| Veggie Omelette | 3 | 0 | Ready to cook immediately |
| Chicken Curry | 3 | 1 (Curry Powder) | Missing 1 ingredient |
| Tomato Pasta | 3 | 1 (Pasta) | Missing 1 ingredient |
| Chicken Salad | 2 | 1 (Lettuce) | Missing 1 ingredient |

---

## 3. The Problem-Solving Journey & Anomaly Resolution

### 3.1 Functional Dependencies Summary

- `recipe_id → recipe_name, category_id, instructions`
- `category_id → category_name`
- `ingredient_id → ingredient_name`
- `unit_id → unit_name`
- `(recipe_id, ingredient_id) → amount, unit_id`

In the final 3NF schema, every determinant is a superkey of its respective table, fulfilling 3NF and BCNF standards.

### 3.2 Real Anomalies Encountered in Early Designs

| Anomaly Type | How It Showed Up in Early Designs | How the 3NF Schema Fixes It |
|---|---|---|
| **Update Anomaly** | "Tomato" was repeated across multiple recipe rows. Renaming it to "Tomatoes" in one row but missing another left the database inconsistent, causing search queries to fail. | `Ingredients.ingredient_name` exists in only one row. Updating it once reflects everywhere across the application instantly. |
| **Insertion Anomaly** | I couldn't add a newly bought item like "Rice" to the pantry or create a "Dessert" category without first attaching it to an existing recipe row. | Catalog tables (`Ingredients`, `Categories`, `Pantry`) operate independently of recipe creation. |
| **Deletion Anomaly** | Deleting the "Chicken Salad" recipe wiped out the only record that "Lettuce" existed. Deleting "Veggie Omelette" erased the "Breakfast" category entirely. | Deleting a recipe only clears link records in `Recipe_Ingredients` via `ON DELETE CASCADE`. Master catalog data remains untouched. |
| **Redundancy** | Full recipe descriptions and category titles were repeated on every single ingredient line in the 1NF table. | Recipe metadata is stored once in `Recipes`, eliminating duplicate text across rows. |

### 3.3 Reflection on Troubleshooting & Design Process

I started this process by actively trying to break my initial Check-in 1 schema with common user scenarios: what happens if I rename an ingredient, add pantry items without cooking, or delete a recipe? Doing those exercises by hand on paper made the three classic anomalies real and concrete rather than just textbook definitions.

My first attempt at fixing things was pulling ingredient names into an Ingredients table while trying to leave quantities there. That broke right away: the quantity isn't a property of Chicken itself—it depends on whether Chicken is being used in Curry (500g) or Salad (200g). Writing out the dependency as `(recipe_id, ingredient_id) → amount` was the moment I realized I needed a true junction table for a many-to-many relationship.

I also considered putting a default unit directly inside the ingredients table (e.g., Garlic → cloves). But since another recipe might call for grams of garlic, I realized units also belong in the junction table, while pulling standard unit names into a lookup table to keep spellings clean.

After finishing the tables, I joined them back together to verify that all 11 original rows were reconstructed perfectly with zero data loss. This troubleshooting process took some iteration, but it resulted in a solid foundation for building out the rest of the application.
