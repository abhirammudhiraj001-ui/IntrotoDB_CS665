# Project Check-in 1 — CS-665

## Recipe Finder: Ingredient-Based Cooking App

**Abhiram Golkonda** · B855Q929

---

## 1. Problem Definition and Mobile Scope

### 1.1 Problem Definition

Many students and individuals struggle to decide what to cook using the ingredients they already have. This often leads to wasted food or unnecessary grocery purchases. This project proposes a mobile application called **Recipe Finder** that allows users to store recipes and search for recipes based on the ingredients they have available.

### 1.2 Target Users and Platform

The application is intended for students and individuals who cook at home. The target platform is **Android**. I am focusing on a single platform instead of supporting both Android and iOS so that the project stays manageable within the semester.

### 1.3 Application Scope

- Add a recipe
- View recipes
- Store ingredients for each recipe
- Search recipes by ingredient

The application will not include social features (sharing, comments, ratings) or external APIs. All data is stored locally in the app's database, so the semester work can stay focused on the database design and the queries that power the main features.

---

## 2. Initial Database Design

For Check-in 1, I am starting with two tables: **Recipes** and **Ingredients**. I plan to expand and normalize this design in Check-in 2 after looking for dependencies and problems in this first version.

### Recipes

| Column | Key | Description |
|---|---|---|
| `recipe_id` | **Primary Key** | Unique ID for a recipe |
| `recipe_name` | | Name of the recipe |
| `instructions` | | Cooking steps |

### Ingredients

| Column | Key | Description |
|---|---|---|
| `ingredient_id` | **Primary Key** | Unique ID for an ingredient entry |
| `recipe_id` | **Foreign Key** | Recipe this ingredient belongs to |
| `ingredient_name` | | Name of the ingredient |
| `quantity` | | Amount needed (for example, 500 g or 2 tbsp) |

**Relationship:** one-to-many. One recipe can have multiple ingredients, while each row in Ingredients belongs to exactly one recipe. The `recipe_id` in Ingredients is a foreign key that references `recipe_id` in Recipes.

---

## 3. SQL and Relational Algebra

### 3.1 Database Creation

```sql
CREATE TABLE Recipes (
    recipe_id    INT PRIMARY KEY,
    recipe_name  VARCHAR(255) NOT NULL,
    instructions TEXT
);

CREATE TABLE Ingredients (
    ingredient_id   INT PRIMARY KEY,
    recipe_id       INT NOT NULL,
    ingredient_name VARCHAR(255) NOT NULL,
    quantity        VARCHAR(50),
    FOREIGN KEY (recipe_id) REFERENCES Recipes(recipe_id)
);
```

The primary keys uniquely identify each row, and the foreign key links every ingredient to a recipe. `NOT NULL` prevents an ingredient from existing without a recipe or a name. Quantity is stored as text because amounts mix numbers and units (such as "2 tbsp" or "1 head"); I may revisit this in Check-in 2.

### Sample Data

```sql
INSERT INTO Recipes (recipe_id, recipe_name, instructions) VALUES
(1, 'Chicken Curry',   'Saute onion, add chicken and curry powder, simmer.'),
(2, 'Tomato Pasta',    'Boil pasta, cook tomato and garlic, combine.'),
(3, 'Veggie Omelette', 'Beat eggs, add onion and tomato, cook in a pan.'),
(4, 'Chicken Salad',   'Chop cooked chicken, mix with lettuce.');

INSERT INTO Ingredients (ingredient_id, recipe_id, ingredient_name, quantity) VALUES
(1,  1, 'Chicken',      '500 g'),
(2,  1, 'Onion',        '1'),
(3,  1, 'Curry Powder', '2 tbsp'),
(4,  2, 'Pasta',        '200 g'),
(5,  2, 'Tomato',       '3'),
(6,  2, 'Garlic',       '2 cloves'),
(7,  3, 'Eggs',         '3'),
(8,  3, 'Onion',        '1'),
(9,  3, 'Tomato',       '1'),
(10, 4, 'Chicken',      '200 g'),
(11, 4, 'Lettuce',      '1 head');
```

---

### 3.2 Query 1: View All Recipes

**Relational Algebra**

$$
\pi_{\text{recipe\_name}}(\text{Recipes})
$$

Plain text: `π_recipe_name (Recipes)`

**SQL**

```sql
SELECT recipe_name
FROM Recipes;
```

> **Expected result:** Chicken Curry, Tomato Pasta, Veggie Omelette, Chicken Salad.
> Projection keeps only the `recipe_name` column, which is what the recipe list screen needs.

---

### 3.3 Query 2: Get Ingredients for a Recipe

**Relational Algebra**

$$
\sigma_{\text{recipe\_id} = 1}(\text{Ingredients})
$$

Plain text: `σ_recipe_id = 1 (Ingredients)`

**SQL**

```sql
SELECT *
FROM Ingredients
WHERE recipe_id = 1;
```

> **Expected result:** Chicken (500 g), Onion (1), Curry Powder (2 tbsp).
> Selection keeps only the rows that belong to recipe 1, which supports the recipe detail screen.

---

### 3.4 Query 3: Join Recipes and Ingredients

**Relational Algebra**

$$
\pi_{\text{recipe\_name},\ \text{ingredient\_name}}(\text{Recipes} \bowtie \text{Ingredients})
$$

Plain text: `π_recipe_name, ingredient_name (Recipes ⋈ Ingredients)`

**SQL**

```sql
SELECT Recipes.recipe_name, Ingredients.ingredient_name
FROM Recipes
JOIN Ingredients
    ON Recipes.recipe_id = Ingredients.recipe_id;
```

> **Expected result:** 11 rows, one for each ingredient, each paired with its recipe name (for example, Chicken Curry with Chicken, Chicken Curry with Onion, and so on).
> The natural join matches rows on the shared `recipe_id` column, and projection keeps just the two names. This query shows the relationship between the two tables.

---

### 3.5 Query 4: Find Recipes That Use Chicken

**Relational Algebra**

$$
\pi_{\text{recipe\_name}}\left(\sigma_{\text{ingredient\_name} = \text{'Chicken'}}(\text{Recipes} \bowtie \text{Ingredients})\right)
$$

Plain text: `π_recipe_name (σ_ingredient_name = 'Chicken' (Recipes ⋈ Ingredients))`

**SQL**

```sql
SELECT recipe_name
FROM Recipes
JOIN Ingredients
    ON Recipes.recipe_id = Ingredients.recipe_id
WHERE ingredient_name = 'Chicken';
```

> **Expected result:** Chicken Curry, Chicken Salad.
> This is the core feature of the app: the join connects each recipe to its ingredients, selection keeps only the Chicken rows, and projection returns the recipe names.

---

### 3.6 Query 5: Recipes With More Than Two Ingredients

**Relational Algebra**

$$
\sigma_{cnt > 2}\left({}_{\text{recipe\_id}}\gamma_{COUNT(*) \rightarrow cnt}(\text{Ingredients})\right)
$$

Plain text: `σ_cnt > 2 ( recipe_id γ COUNT(*) → cnt (Ingredients) )`

**SQL**

```sql
SELECT recipe_id, COUNT(*) AS cnt
FROM Ingredients
GROUP BY recipe_id
HAVING COUNT(*) > 2;
```

> **Expected result:** `recipe_id` 1, 2, and 3 (three ingredients each). Recipe 4 has only two ingredients, so it is filtered out.
> Aggregation first groups the ingredients by recipe and counts them; the selection applied afterward (`HAVING` in SQL) keeps only groups whose count is above 2. Here γ is written with the grouping attribute (`recipe_id`) on the left and the aggregate on the right.

---

## 4. AI Utilization Plan

I plan to use two generative AI agents, **ChatGPT** and **Claude**, as learning and debugging tools, not as a way to generate the full project. My goal is to understand every part of my database and app well enough to explain and change it myself.

- **ChatGPT:** my main tool for SQL debugging and for understanding Android database code while I build the app.
- **Claude:** my second tool for explaining relational algebra concepts and reviewing my schema design. I will also use it to compare explanations with ChatGPT when I am unsure about an answer, so I do not rely on a single source.

### Example Uses and Prompts

**Learning relational algebra (Claude)**

> "Explain the difference between selection, projection, and join in relational algebra using a small recipe database. How do I decide which operator to use first?"

**SQL debugging (ChatGPT)**

> "I am getting a foreign key error when inserting into Ingredients. Explain what the error means and what I should check before showing me a fix."

**Database design (Claude)**

> "Review my two-table recipe schema and point out where data might be repeated or where problems could appear when I update or delete rows. Don't redesign it for me yet."

**Understanding Android/database code (ChatGPT)**

> "Explain this Android database code line by line so I understand how the app talks to the database before I change anything."

I will test every SQL query and piece of code myself, compare results against the expected output, and check documentation when needed. I will also keep a short log of meaningful AI interactions and what I learned from each one.
