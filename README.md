# Recipe Rating Prediction

**Author:** Xuan Yin

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

We performed the following cleaning steps on the raw data:

1. **Left merged** `RAW_recipes.csv` with `RAW_interactions.csv` on recipe ID. A left merge ensures every recipe is retained even if it has no reviews — dropping unrated recipes would bias our dataset toward only popular ones.
2. **Replaced ratings of 0 with `NaN`** — on Food.com, a rating of 0 means the user submitted a review without selecting a star rating. It is not a genuine score, so including it as 0 would artificially pull average ratings downward.
3. **Computed average rating per recipe** (`avg_rating`) and the number of valid ratings (`n_ratings`) and added both back to the recipes dataframe.
4. **Parsed the `nutrition` column** from a string representation into separate numeric columns: `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `sat_fat`, and `carbs`. These values are expressed as percentage of daily value (PDV), except calories which is an absolute count.
5. **Capped `minutes` at the 99th percentile** (730 minutes) to remove extreme outliers — some recipes had cook times in the millions of minutes, which are clearly data entry errors and would distort visualizations and model training.

Here are the first 5 rows of the cleaned dataframe (relevant columns shown):

| name | minutes | n_steps | n_ingredients | calories | avg_rating | n_ratings |
|------|---------|---------|---------------|----------|------------|-----------|
| 1 brownies in the world best ever | 40 | 10 | 9 | 138.4 | 4.0 | 1 |
| 1 in canada chocolate chip cookies | 45 | 12 | 11 | 595.1 | 5.0 | 1 |
| 412 broccoli casserole | 40 | 6 | 9 | 194.8 | 5.0 | 3 |
| millionaire pound cake | 120 | 7 | 7 | 878.3 | 5.0 | 1 |
| 2000 meatloaf | 90 | 17 | 13 | 267.0 | 5.0 | 2 |

---

### Univariate Analysis

**Distribution of Average Recipe Ratings**

<iframe src="assets/rating_dist.html" width="800" height="500" frameborder="0"></iframe>

The distribution of average ratings is heavily left-skewed — 47,784 out of 83,782 recipes have a perfect average rating of 5.0, and the vast majority of recipes fall between 4 and 5 stars. This class imbalance is a critical observation: it means a naive model that always predicts 5 would already achieve high accuracy, which is why we use F1 score instead of accuracy to evaluate our classifier.

**Distribution of Number of Steps**

<iframe src="assets/steps_dist.html" width="800" height="500" frameborder="0"></iframe>

The number of steps per recipe is right-skewed, with a median of 9 steps and a mean of about 10. Most recipes fall between 6 and 15 steps, though a small number of recipes have over 30 steps. This suggests that most Food.com recipes are moderately complex.

---

### Bivariate Analysis

**Cooking Time vs. Average Rating**

<iframe src="assets/time_vs_rating.html" width="800" height="500" frameborder="0"></iframe>

There is no clear linear relationship between cooking time and average rating. Recipes across all cooking times tend to cluster around 4–5 stars, consistent with the overall rating distribution. This suggests that how long a recipe takes to make does not strongly predict how users will rate it.

**Number of Steps vs. Average Rating**

<iframe src="assets/steps_vs_rating.html" width="800" height="500" frameborder="0"></iframe>

Similarly, there is no strong trend between number of steps and average rating. Both simple and complex recipes receive high ratings, suggesting that recipe complexity alone is not a reliable predictor of user satisfaction.

---

### Interesting Aggregates

We grouped recipes into bins by number of steps and computed the mean average rating for each group:

| Steps | Mean Avg Rating | Recipe Count |
|-------|----------------|--------------|
| 1–5   | 4.638 | 18,547 |
| 6–10  | 4.616 | 31,915 |
| 11–15 | 4.618 | 18,357 |
| 16–20 | 4.638 | 7,410 |
| 20+   | 4.647 | 4,944 |

<iframe src="assets/steps_agg.html" width="800" height="500" frameborder="0"></iframe>

Interestingly, the mean average rating is remarkably consistent across all step count groups, ranging only from 4.616 to 4.647. This tells us that users do not systematically rate simpler or more complex recipes higher — the number of steps alone carries very little signal for predicting ratings. This motivates us to look beyond simple structural features and incorporate nutritional information in our model.

---

## Assessment of Missingness

### NMAR Analysis

We believe that the `description` column in the dataset is likely **NMAR** (Not Missing At Random). A recipe's description is missing not randomly, but likely because the contributor chose not to write one — and that decision may be related to the content of the description itself. For example, contributors who feel their recipe is self-explanatory, or who are less experienced users on Food.com, may be more likely to skip writing a description. The missingness is therefore tied to the unobserved value itself, which is the defining characteristic of NMAR.

To determine whether this missingness is actually MAR (Missing At Random), we would need additional data — for example, the contributor's account age, their total number of submitted recipes, or their activity level on the platform. If less active or newer contributors are more likely to skip descriptions, then the missingness would depend on an observed variable and could be reclassified as MAR.

---

### Missingness Dependency

We analyzed the missingness of `avg_rating`, which is missing for **2,609 out of 83,782 recipes (3.1%)**. These are recipes that exist on Food.com but have never received a rating from any user. We ran permutation tests to determine whether this missingness depends on other columns, using the **absolute difference in group means** as our test statistic and a significance level of **0.05**.

---

#### Does missingness of `avg_rating` depend on `n_ingredients`?

**Null Hypothesis:** The missingness of `avg_rating` does not depend on the number of ingredients in the recipe. Any observed difference in mean `n_ingredients` between recipes with and without a missing rating is due to random chance.

**Alternative Hypothesis:** The missingness of `avg_rating` does depend on the number of ingredients. Recipes with missing ratings have a systematically different number of ingredients than recipes with ratings.

**Test Statistic:** Absolute difference in mean `n_ingredients` between recipes where `avg_rating` is missing and recipes where it is not.

**Significance Level:** 0.05

The plot below shows the distribution of `n_ingredients` for recipes where `avg_rating` is missing versus not missing:

<iframe src="assets/missing_dist.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;"></iframe>

Recipes with missing ratings appear to have slightly more ingredients on average. We ran a permutation test to assess whether this difference is statistically significant:

<iframe src="assets/missing_ingr.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;"></iframe>

The observed difference in mean number of ingredients was **0.2542**. After 1,000 permutations, we obtained a **p-value of 0.001**, which is less than our significance level of 0.05. We **reject the null hypothesis** — the missingness of `avg_rating` does depend on `n_ingredients`. This makes intuitive sense: recipes with more ingredients may be more niche or complex, attracting fewer users and therefore fewer ratings.

---

#### Does missingness of `avg_rating` depend on `protein`?

**Null Hypothesis:** The missingness of `avg_rating` does not depend on the protein content of the recipe. Any observed difference in mean protein between recipes with and without a missing rating is due to random chance.

**Alternative Hypothesis:** The missingness of `avg_rating` does depend on the protein content of the recipe.

**Test Statistic:** Absolute difference in mean `protein` (PDV) between recipes where `avg_rating` is missing and recipes where it is not.

**Significance Level:** 0.05

<iframe src="assets/missing_prot.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;"></iframe>

The observed difference in mean protein content was **1.2873**. After 1,000 permutations, we obtained a **p-value of 0.201**, which is greater than our significance level of 0.05. We **fail to reject the null hypothesis** — the missingness of `avg_rating` does not depend on protein content. This makes intuitive sense: whether a recipe gets rated by users is unlikely to be related to how much protein it contains. The protein content is a property of the recipe itself, not a factor that would influence user engagement.

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