# What Makes a Recipe a 5-Star Hit?

**Author:** Xuan Yin

---

## Introduction

What makes a recipe highly rated? Every home cook has wondered why some dishes earn rave reviews while others fall flat. Understanding what separates a 5-star recipe from a mediocre one has real practical value for anyone looking to improve their cooking or share their recipes with the world.

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

<iframe src="assets/rating_dist.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;"></iframe>

The distribution of average ratings is heavily left-skewed — 47,784 out of 83,782 recipes have a perfect average rating of 5.0, and the vast majority of recipes fall between 4 and 5 stars. This class imbalance is a critical observation: it means a naive model that always predicts 5 would already achieve high accuracy, which is why we use F1 score instead of accuracy to evaluate our classifier.

**Distribution of Number of Steps**

<iframe src="assets/steps_dist.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;"></iframe>

The number of steps per recipe is right-skewed, with a median of 9 steps and a mean of about 10. Most recipes fall between 6 and 15 steps, though a small number of recipes have over 30 steps. This suggests that most Food.com recipes are moderately complex.

---

### Bivariate Analysis

**Cooking Time vs. Average Rating**

<iframe src="assets/time_vs_rating.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;"></iframe>

There is no clear linear relationship between cooking time and average rating. Recipes across all cooking times tend to cluster around 4–5 stars, consistent with the overall rating distribution. This suggests that how long a recipe takes to make does not strongly predict how users will rate it.

**Number of Steps vs. Average Rating**

<iframe src="assets/steps_vs_rating.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;"></iframe>

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

<iframe src="assets/steps_agg.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;" ></iframe>

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

### Do more complex recipes receive lower ratings than simpler recipes?

From our EDA, we observed that mean ratings across recipes with different numbers of steps were remarkably similar. We now formally test whether this difference is statistically significant or simply due to random chance.

We define **"complex" recipes** as those with more than 9 steps (above the median of 9 steps), and **"simple" recipes** as those with 9 steps or fewer.

**Null Hypothesis:** Complex recipes and simple recipes are rated on the same scale. Any observed difference in mean ratings between the two groups is due to random chance.

**Alternative Hypothesis:** Complex recipes receive lower average ratings than simple recipes.

**Test Statistic:** Difference in mean average rating (complex − simple). We chose this directional test statistic because our alternative hypothesis is one-sided — we are specifically testing whether complex recipes are rated *lower*.

**Significance Level:** 0.05

We ran a permutation test with 1,000 permutations, shuffling the `avg_rating` values across both groups each time to simulate the null hypothesis.

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;"></iframe>

| | Mean Rating |
|---|---|
| Simple recipes (≤ 9 steps) | 4.6268 |
| Complex recipes (> 9 steps) | 4.6237 |
| Observed difference | -0.0031 |
| P-value | 0.235 |

### Conclusion

Since the p-value of **0.235** is greater than our significance level of 0.05, we **fail to reject the null hypothesis**. The data does not provide sufficient evidence that complex recipes receive lower ratings than simple ones. The observed difference of -0.0031 is well within the range of what we would expect by random chance alone.

This is an important finding for our prediction problem: recipe complexity as measured by number of steps carries very little signal for predicting ratings. This motivates us to look beyond structural features like steps and cooking time, and instead incorporate nutritional information and other recipe characteristics in our model.

---

## Framing a Prediction Problem

We frame our prediction problem as follows: **given a recipe's characteristics, can we predict its average star rating?**

This is a **multi-class classification problem**. We round `avg_rating` to the nearest integer, giving us five possible classes: 1, 2, 3, 4, and 5 stars. We chose classification over regression because star ratings are naturally ordinal categories, and it allows us to use a more meaningful evaluation metric given the class imbalance in our data.

**Response variable:** `avg_rating` rounded to the nearest integer. We chose this as our response variable because it directly answers our central question — what makes a recipe highly rated? — and is a natural representation of user satisfaction on a 1–5 scale.

**Features used at prediction time:** We only use features that describe the recipe itself and would be known *before* any user rates it: `n_steps`, `n_ingredients`, `minutes`, `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `sat_fat`, and `carbs`. We deliberately exclude `n_ratings` since that is derived from user interactions and would not be available at prediction time for a brand new recipe.

**Evaluation metric:** Weighted F1 score. As we observed in EDA, over half of all recipes have a perfect 5-star average rating — this severe class imbalance means accuracy would be misleading. A model that always predicts 5 would achieve high accuracy while learning nothing useful. Weighted F1 score accounts for class imbalance by weighting each class's F1 score by its support, giving us a more honest picture of true model performance across all rating categories.

---

## Baseline Model

Our baseline model uses a **Random Forest Classifier** with two quantitative features:

- `n_steps` — number of steps in the recipe (quantitative)
- `n_ingredients` — number of ingredients required (quantitative)

Both features are standardized using `StandardScaler` before being passed to the classifier. All steps are implemented in a single `sklearn` Pipeline. Neither feature required ordinal or one-hot encoding since both are already numeric.

### Results

| Metric | Score |
|--------|-------|
| Weighted F1 Score | 0.5593 |

| Class | Precision | Recall | F1 |
|-------|-----------|--------|----|
| 1 star | 0.00 | 0.00 | 0.00 |
| 2 stars | 0.00 | 0.00 | 0.00 |
| 3 stars | 0.00 | 0.00 | 0.00 |
| 4 stars | 0.19 | 0.00 | 0.00 |
| 5 stars | 0.69 | 1.00 | 0.81 |

### Analysis

The baseline model is **not good**. Despite a weighted F1 of 0.5593, the classification report reveals that the model essentially predicts 5 stars for almost every recipe — achieving perfect recall on class 5 while completely failing on classes 1 through 4. This behavior is a direct consequence of the severe class imbalance we identified in EDA: 56,124 out of 81,173 recipes (69%) have a 5-star rounded average rating.

This tells us that `n_steps` and `n_ingredients` alone carry almost no discriminative signal for predicting rating. To build a meaningful model, we need to incorporate more informative features — particularly nutritional information — and address the class imbalance more carefully. This motivates our Final Model in Step 7.

---

## Final Model

### Feature Engineering

We added four new features on top of the baseline's `n_steps` and `n_ingredients`:

| Feature | Transformation | Justification |
|---------|---------------|---------------|
| `calories` | QuantileTransformer | Caloric content is heavily right-skewed with extreme outliers. QuantileTransformer maps it to a normal distribution, reducing the influence of extreme values. Indulgent high-calorie recipes tend to be comfort foods that users rate highly. |
| `protein` | QuantileTransformer | Protein content is similarly skewed. Health-conscious users may rate high-protein recipes more favorably, making this a meaningful signal. |
| `sugar` | QuantileTransformer | Our hypothesis test found that sugary recipes tend to be rated slightly lower. Including this feature lets the model capture that relationship. |
| `minutes_capped` | StandardScaler | Cook time is roughly normally distributed after capping at the 99th percentile. Very long recipes may frustrate users and lead to lower ratings. |

### Hyperparameter Tuning

We tuned two hyperparameters of the `RandomForestClassifier` using 5-fold cross-validation with `GridSearchCV`:

- **`max_depth`** — controls how deep each tree grows. Shallower trees underfit; deeper trees overfit. We searched `[10, 20, 30, None]`.
- **`n_estimators`** — number of trees in the forest. More trees produce more stable predictions but increase training time. We searched `[100, 200]`.

We also set `class_weight="balanced"` to address class imbalance — this adjusts each class's weight inversely proportional to its frequency, encouraging the model to learn minority classes (1–3 stars) rather than always predicting 5.

The best hyperparameters found were `max_depth=20` and `n_estimators=100`, with a cross-validated F1 of **0.5830**.

### Results

| Metric | Baseline | Final Model |
|--------|----------|-------------|
| Weighted F1 Score | 0.5593 | 0.5769 |
| Improvement | — | +0.0176 |

| Class | Precision | Recall | F1 |
|-------|-----------|--------|----|
| 1 star | 0.00 | 0.00 | 0.00 |
| 2 stars | 0.00 | 0.00 | 0.00 |
| 3 stars | 0.08 | 0.00 | 0.00 |
| 4 stars | 0.29 | 0.10 | 0.14 |
| 5 stars | 0.69 | 0.91 | 0.79 |

### Analysis

The final model improves upon the baseline with a weighted F1 score of **0.5769** compared to **0.5593**. More importantly, the model now correctly identifies some 4-star recipes (recall of 0.10), whereas the baseline completely ignored all non-5-star classes. The addition of nutritional features and `class_weight="balanced"` helped the model begin to distinguish between rating classes.

The persistent difficulty in predicting 1–3 star ratings reflects a fundamental challenge in this dataset: with only 590, 775, and 2,760 recipes in those classes respectively compared to 56,124 five-star recipes, even a well-tuned model struggles to learn meaningful patterns for rare classes. This is an inherent property of the data generating process — Food.com users overwhelmingly rate recipes positively.

---

## Fairness Analysis

### Does our model perform differently for high-calorie vs. low-calorie recipes?

We split recipes in our test set into two groups based on calorie content:
- **Low-calorie:** recipes with calories ≤ 303.80 (median calories in test set)
- **High-calorie:** recipes with calories > 303.80

We chose this split because calories is one of our key features, and we want to ensure the model does not systematically perform better or worse for recipes at different ends of the calorie spectrum.

**Evaluation metric:** Weighted F1 score (same metric used to evaluate overall model performance).

**Null Hypothesis:** The model is fair. Its weighted F1 score for low-calorie and high-calorie recipes are roughly the same, and any observed difference is due to random chance.

**Alternative Hypothesis:** The model is unfair. Its weighted F1 score differs between low-calorie and high-calorie recipes.

**Test Statistic:** Difference in weighted F1 score (low-calorie − high-calorie).

**Significance Level:** 0.05

We ran a permutation test with 1,000 permutations, shuffling the group labels each time to simulate the null hypothesis.

<iframe src="assets/fairness.html" width="800" height="500" frameborder="0" style="background-color: #ECE4D8;"></iframe>

| Group | Weighted F1 |
|-------|------------|
| Low-calorie (≤ 303.80 cal) | 0.5807 |
| High-calorie (> 303.80 cal) | 0.5709 |
| Observed difference | 0.0098 |
| P-value | 0.2730 |

### Conclusion

Since the p-value of **0.273** is greater than our significance level of 0.05, we **fail to reject the null hypothesis**. The observed difference of 0.0098 in weighted F1 between low-calorie and high-calorie recipes is well within the range of what we would expect by random chance alone. Our model appears to perform equally well across both calorie groups, suggesting it does not exhibit calorie-based bias in its predictions.