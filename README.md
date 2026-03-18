# Recipe Rating Prediction

**Authors:** [Your Name]

---

## Introduction

What makes a recipe highly rated? With millions of home cooks turning to the internet for culinary inspiration, understanding what separates a 5-star recipe from a mediocre one has real practical value — both for recipe creators and for platforms like Food.com that want to surface the best content.

This project investigates a dataset of recipes and user interactions scraped from Food.com, originally compiled for a 2008 research paper on personalized recipe recommendation, [Generating Personalized Recipes from Historical User Preferences](https://cseweb.ucsd.edu/~jmcauley/pdfs/emnlp19c.pdf) by Majumder et al.

**Central Question: What recipe characteristics best predict a recipe's average rating, and can we build a model to predict it?**

Answering this question helps us understand what features — complexity, cook time, and nutritional content — are most associated with user satisfaction, and could help recipe contributors on Food.com improve their recipes to better align with what users enjoy.

The dataset contains **83,782 recipes** submitted since 2008. We merged the recipes with their user interactions and computed the average rating per recipe. The columns most relevant to our question are:

| Column | Description |
|--------|-------------|
| `avg_rating` | Average star rating for the recipe (1–5); our prediction target |
| `minutes` | Total preparation and cooking time in minutes |
| `n_steps` | Number of steps in the recipe instructions |
| `n_ingredients` | Number of ingredients required |
| `calories` | Total calories (parsed from `nutrition`) |
| `total_fat` | Total fat as % daily value |
| `sugar` | Sugar as % daily value |
| `sodium` | Sodium as % daily value |
| `protein` | Protein as % daily value |
| `sat_fat` | Saturated fat as % daily value |
| `carbs` | Carbohydrates as % daily value |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

We performed the following cleaning steps:

1. **Left merged** `RAW_recipes.csv` with `RAW_interactions.csv` on recipe ID, so every recipe is kept even if it has no reviews.
2. **Replaced ratings of 0 with `NaN`** — on Food.com, a rating of 0 means the user left a review without submitting a star rating. It is a missing value, not a genuine score, so treating it as 0 would bias our averages downward.
3. **Computed average rating per recipe** and added it back to the recipes dataframe as `avg_rating`.
4. **Parsed the `nutrition` column** from a string representation of a list into separate numeric columns: `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `sat_fat`, and `carbs`.
5. **Added `n_ratings`** — the number of valid (non-NaN) ratings per recipe. This captures how much user engagement a recipe received and may be useful as a feature.

Here are the first 5 rows of the cleaned dataframe (selected relevant columns):

| name | minutes | n_steps | n_ingredients | calories | avg_rating | n_ratings |
|------|---------|---------|---------------|----------|------------|-----------|
| 1 brownies in the world best ever | 40 | 10 | 9 | 138.4 | 4.0 | 1 |
| 1 in canada chocolate chip cookies | 45 | 12 | 11 | 595.1 | 5.0 | 1 |
| 412 broccoli casserole | 40 | 6 | 9 | 194.8 | 5.0 | 3 |
| millionaire pound cake | 120 | 7 | 7 | 878.3 | 5.0 | 1 |
| 2000 meatloaf | 90 | 17 | 13 | 267.0 | 5.0 | 2 |

### Univariate Analysis

**[INSERT PLOTLY HISTOGRAM OF avg_rating HERE]**

The distribution of average ratings is heavily left-skewed, with the vast majority of recipes rated 4 or 5 stars. This class imbalance is important to keep in mind when building our model — it means accuracy alone would be a misleading metric.

**[INSERT PLOTLY HISTOGRAM OF n_steps HERE]**

The distribution of number of steps is right-skewed, with most recipes having between 5 and 15 steps. A small number of recipes have extremely high step counts.

### Bivariate Analysis

**[INSERT PLOTLY SCATTER OR BOX PLOT OF avg_rating vs n_steps HERE]**

When we examine average rating against number of steps, we see no strong linear trend — recipes with both few and many steps receive high ratings. This suggests that complexity alone does not drive user satisfaction.

### Interesting Aggregates

We grouped recipes into bins by number of steps and computed the mean average rating for each bin:

**[INSERT PLOTLY BAR CHART OF avg rating by steps bin HERE]**

| Steps | Mean Avg Rating |
|-------|----------------|
| 1–5   | **[value]** |
| 6–10  | **[value]** |
| 11–15 | **[value]** |
| 16–20 | **[value]** |
| 20+   | **[value]** |

---

## Assessment of Missingness

*Coming soon.*

---

## Hypothesis Testing

*Coming soon.*

---

## Framing the Prediction Problem

*Coming soon.*

---

## Baseline Model

*Coming soon.*

---

## Final Model

*Coming soon.*

---

## Fairness Analysis

*Coming soon.*