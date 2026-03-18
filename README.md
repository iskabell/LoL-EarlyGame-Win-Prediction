#   Rift to Result: Predicting Wins in League of Legends
By Isabel Yang

Machine learning model predicting early-game match outcomes in League of Legends. Final project for DSC 80 at UCSD.

## Introduction

This project uses professional League of Legends match data from Oracle’s Elixir. The dataset contains detailed statistics about in-game performance, which makes it useful for studying how early-game advantages relate to match outcomes.

When first exploring the dataset, I considered several possible questions, including whether being ahead at 15 minutes increases a team’s chance of winning, which early-game features are most associated with match outcome, and whether objective advantages like first blood or first tower are strong predictors of success. The question I ultimately chose to investigate is:

**Does being ahead in the early game increase a team’s probability of winning?**

This question is interesting because early-game advantages such as gold lead, XP lead, and first objectives are often seen as very important in competitive League of Legends, but I wanted to examine how strongly those factors are actually associated with winning in the data. Readers should care because this project helps explain whether early momentum truly matters and which early indicators are most useful for understanding and predicting match results.

After filtering to team-level observations, the dataset contains 18,472 rows and 8 columns. The most relevant columns for this question are:

- `result`: whether the team won or lost the match
- `golddiffat15`: gold difference at 15 minutes
- `xpdiffat15`: experience difference at 15 minutes
- `killsat15`: team kills at 15 minutes
- `opp_killsat15`: opponent kills at 15 minutes
- `firstblood`: whether the team got first blood
- `firsttower`: whether the team got first tower
- `side`: whether the team played on red side or blue side

Together, these columns capture different types of early-game advantage, including economy, experience, combat performance, and objective control.

## Data Cleaning and Exploratory Data Analysis

The original Oracle’s Elixir dataset contains match statistics at multiple levels of observation, including individual players and teams. Since my project question is about whether **a team’s early-game advantage increases its probability of winning**, I first filtered the dataset to rows where `position == "team"`. This step is important because the data generating process records both player-level and team-level information, and using player rows would not match the unit of analysis for my question. After filtering, each row represented a single **team-game observation**.

Next, I created a new feature called `kill_diff_15`, defined as a team’s kills at 15 minutes minus its opponent’s kills at 15 minutes. This engineered variable better captures relative early-game combat advantage than raw kills alone, since winning in League of Legends depends on how a team performs compared to its opponent, not just its absolute total.

I then selected the columns most relevant to early-game performance and match outcome: `result`, `side`, `golddiffat15`, `xpdiffat15`, `csdiffat15`, `kill_diff_15`, `firstblood`, `firstdragon`, and `firsttower`. Finally, I dropped rows with missing values in these columns so that the exploratory analysis and later predictive models would be based on complete observations only. After cleaning, the resulting DataFrame contained **18,472 rows** and **9 columns**.

Here is the head of the cleaned DataFrame:

| result | side | golddiffat15 | xpdiffat15 | kill_diff_15 | firstblood | firstdragon | firsttower |
|---:|:---|---:|---:|---:|---:|---:|---:|
| 0 | Blue | -3837.0 | -469.0 | -3.0 | 0.0 | 0.0 | 0.0 |
| 1 | Red  | 3837.0  | 469.0  | 3.0  | 1.0 | 1.0 | 1.0 |
| 1 | Blue | 5069.0  | 2014.0 | 5.0  | 1.0 | 0.0 | 1.0 |
| 0 | Red  | -5069.0 | -2014.0| -5.0 | 0.0 | 1.0 | 0.0 |
| 0 | Blue | 118.0   | 1990.0 | 4.0  | 0.0 | 0.0 | 0.0 |

### Univariate Analysis

<iframe
  src="assets/gold-diff-hist.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

This histogram shows the distribution of `golddiffat15`, the gold difference at 15 minutes. The distribution is centered near 0, which makes sense because one team’s gold lead is the other team’s deficit, but the wide spread shows that some matches already have very large early-game advantages by the 15-minute mark.

<iframe
  src="assets/xp-diff-hist.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

This histogram shows the distribution of `xpdiffat15`, the experience difference at 15 minutes. The distribution is centered near 0 because one team’s advantage is the other team’s deficit, while the wide spread suggests that teams can have substantially different early-game experience leads across matches.

### Bivariate Analysis

<iframe
  src="assets/xp-vs-gold-scatter.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

This scatterplot shows a strong positive relationship between gold difference and XP difference at 15 minutes. Teams with positive gold and XP advantages tend to cluster more heavily among wins, which suggests that multiple forms of early-game advantage often occur together and are strongly associated with match outcome.

<iframe
  src="assets/gold-diff-by-result.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

This plot compares gold difference at 15 minutes across wins and losses. Winning teams tend to have noticeably higher gold differences than losing teams, suggesting that early economic advantage is strongly associated with match outcome.

### Interesting Aggregates

One useful grouped summary is the win rate across different buckets of gold difference at 15 minutes:

| gold_bucket | result |
|:--|--:|
| Large Deficit | 0.12 |
| Small Deficit | 0.33 |
| Even | 0.50 |
| Small Lead | 0.67 |
| Large Lead | 0.88 |

This table shows a very clear pattern: as gold difference at 15 minutes increases, win rate also increases. Teams with a large deficit win only about **12%** of the time, while teams with a large lead win about **88%** of the time, which strongly supports the idea that early-game advantage is closely tied to eventual victory.

A second grouped summary shows win rate by both `firsttower` and gold-difference bucket:

| firsttower | Large Deficit | Small Deficit | Even | Small Lead | Large Lead |
|---:|---:|---:|---:|---:|---:|
| 0.0 | 0.11 | 0.28 | 0.41 | 0.57 | 0.78 |
| 1.0 | 0.22 | 0.43 | 0.59 | 0.72 | 0.89 |

This pivot table suggests that securing first tower is associated with a higher win rate across every gold-difference bucket. Even after accounting for gold lead, teams that take first tower tend to perform better, which suggests that early objective control may provide information beyond pure economy alone.

## Assessment of Missingness

### MNAR Analysis

One column in the original dataset that I believe could be **MNAR** is `playerid`. A value in `playerid` may be missing when Oracle’s Elixir or the underlying data source is unable to confidently match a row to a specific player. This suggests that the missingness may depend on the unobserved value itself. For example, certain players may be harder to identify because they use alternate accounts, inconsistent spellings, name changes, or incomplete roster information. In that case, the missingness is not just a function of other observed columns, but of the missing player identity itself, which is why `playerid` is plausibly **MNAR**.

To better explain this missingness and potentially make it **MAR** instead, I would want additional data about roster history, alternate usernames, player name standardization, and league-level record quality. With those extra variables, it might become possible to explain missing `playerid` values using observed information rather than treating the missingness as depending on the missing value itself.

### Missingness Dependency

For the missingness dependency analysis, I focused on the column `deathsat25`, which records the number of deaths a team had by 25 minutes. This column has non-trivial missingness because some matches end before the 25-minute mark, so 25-minute statistics are not always observed. This is closely tied to the data generating process: if a game does not last long enough, then values measured at 25 minutes cannot be recorded.

To study this missingness, I created a Boolean indicator column called `deathsat25_missing` and used permutation tests to determine whether the missingness of `deathsat25` depends on other variables. I tested whether its missingness depends on `short_game`, which indicates whether a match lasted fewer than 25 minutes, and on `result`, which indicates whether the team won or lost.

For the first permutation test, I used the difference in missingness rates between short and non-short games as the test statistic.

- **Null hypothesis:** The missingness of `deathsat25` does not depend on whether a game is short.
- **Alternative hypothesis:** The missingness of `deathsat25` does depend on whether a game is short.

The observed missingness rates were very different: the missingness rate of `deathsat25` was about **0.08** for games lasting at least 25 minutes and about **0.66** for short games. The permutation test produced a **p-value of 0.0**, so I rejected the null hypothesis. This provides strong evidence that the missingness of `deathsat25` **does depend** on whether a game is short.

For the second permutation test, I again used the difference in missingness rates, this time comparing wins and losses.

- **Null hypothesis:** The missingness of `deathsat25` does not depend on match result.
- **Alternative hypothesis:** The missingness of `deathsat25` does depend on match result.

This test produced a **p-value of 1.0**, so I failed to reject the null hypothesis. There is not enough statistical evidence to conclude that the missingness of `deathsat25` depends on `result`.

Together, these results suggest that the missingness of `deathsat25` is strongly related to match duration, but not to whether the team won or lost. That makes sense from the data generating process: 25-minute statistics are mainly missing because some matches end before 25 minutes, not because of the final outcome itself.

### Missingness Exploration Plot

<iframe
  src="assets/deathsat25-missingness.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

This plot compares the missingness rate of `deathsat25` for short games versus non-short games. The much higher missingness rate among short games supports the conclusion from the permutation test that `deathsat25` missingness depends strongly on whether the game lasted at least 25 minutes.

## Hypothesis Testing

To investigate whether early-game advantage is associated with winning, I performed a permutation test comparing the win rates of two groups: teams that were ahead in gold at 15 minutes and teams that were not ahead in gold at 15 minutes. I created a Boolean variable `ahead15`, where `True` means `golddiffat15 > 0`.

- **Null hypothesis:** Teams that are ahead in gold at 15 minutes and teams that are not ahead at 15 minutes have the same win rate.
- **Alternative hypothesis:** Teams that are ahead in gold at 15 minutes have a higher win rate than teams that are not ahead at 15 minutes.

I used the **difference in win rates** between the two groups as my test statistic, and I used a significance level of **0.05**. This is an appropriate choice because my question is specifically about whether having an early gold lead is associated with a higher probability of winning, so comparing group win rates directly answers that question.

In the observed data, the win rate for teams that were **not ahead** in gold at 15 minutes was about **0.31**, while the win rate for teams that **were ahead** was about **0.73**. This gives an observed difference in win rates of about **0.423**.

After running the permutation test, I obtained a **p-value of 0.0**. Since this is well below 0.05, I reject the null hypothesis. There is strong statistical evidence that teams ahead in gold at 15 minutes tend to win more often than teams that are not ahead at 15 minutes.

This result supports the main question of the project: early-game economic advantage is strongly associated with eventual match outcome. While this does not prove that being ahead in gold directly causes a win, it does show a clear and statistically significant relationship between early gold lead and victory.

