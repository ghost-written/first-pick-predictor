# first-pick-predictor
This is the writeup for a classification-based prediction model which uses the champion ban list and several other features to predict the first champion picked in a League of Legends game.

## Introduction and Question Identification
### Introduction

Our question is simple: **can we predict the first champion picked in a game?** We would like to achieve it using only knowledge of which champions were banned, but **do we need more**?

Most CS or Engineering students play League of Legends at least once in their lives, even the ones who don't admit it. We find it a suitable subject for 'professional interest' due to the complexity and rich variety of the data, but for those more 'amateurly interested', the author assures you he doesn't judge- being himself hardstuck in Gold II for as long as he has played this game. This study uses data provided by <a href='https://oracleselixir.com/'>Oracle's Elixir</a> through Riot Games for its analysis. The data sampled are from the year 2024. Results may not be applicable to years before and after 2024 due to changes in both encouraged and practiced gameplay style.

This dataset has 117,648 rows, though as we will see later, many of these are duplicate for our purposes and will be subject to removal. 

At a baseline, we care about the columns ban1-5, and the column pick1. These ban data should be useful in predicting the first champion picked. While all ban information resides in ban1-5 and all pick information resides in pick1, there is cleaning and rearranging that must be done so as not to repeatedly sample information, and to indicate where the actual first pick resides. Information is separated per-row by participantid. Each player 1-10 has an id 1-10, while game information itself, including pick1-5, resides in the rows where participantid is 100 (for team 1) and 200 (for team 2).

So, the columns we want to look at are:
- **ban1-5:** Each row in each column in the range ban1 to ban5 contains a champion banned by **the team** indicated in that row.
- **pick1:** Each row in pick1 contains the first champion picked by **the team** indicated in that row.
- **gameid:** The id of each game, which may be useful for data alteration later.
- **participantid:** The id of each participant, which will be needed to filter this down to **team rows only**.
- **patch:** The game version, which may be a useful feature later.
- **teamname:** The name of the team, which may be a useful feature later.

## Data Cleaning and Exploratory Data Analysis
### Data Cleaning

Now it's time to clean and analyze the data. As our first step, let's cut the dataset down to the above columns and rows and display it.

| gameid             |   participantid | ban1     | ban2     | ban3         | ban4      | ban5      | pick1        |   patch | teamname    |
|:-------------------|----------------:|:---------|:---------|:-------------|:----------|:----------|:-------------|--------:|:------------|
| 10660-10660_game_1 |             100 | Akali    | Nocturne | K'Sante      | Lee Sin   | Wukong    | Kalista      |   13.24 | LNG Esports |
| 10660-10660_game_1 |             200 | Poppy    | Ashe     | Neeko        | Vi        | Jarvan IV | Renata Glasc |   13.24 | Rare Atom   |
| 10660-10660_game_2 |             100 | Nocturne | Udyr     | Renata Glasc | Nautilus  | Lee Sin   | Neeko        |   13.24 | LNG Esports |
| 10660-10660_game_2 |             200 | Poppy    | Ashe     | Rumble       | Tristana  | Lucian    | Kalista      |   13.24 | Rare Atom   |
| 10660-10660_game_3 |             100 | Rell     | Nocturne | Tristana     | Jarvan IV | Rumble    | Neeko        |   13.24 | LNG Esports |

Not bad, but we can do more. From the individual player rows we just dropped, we can retrieve the **position** column of the player whose champion appears in the pick1 slot. This requires matching and merging, but leaves us with potentially useful information for later.

| gameid             |   participantid | ban1     | ban2     | ban3         | ban4      | ban5      | pick1        |   patch | teamname    | position   |
|:-------------------|----------------:|:---------|:---------|:-------------|:----------|:----------|:-------------|--------:|:------------|:-----------|
| 10660-10660_game_1 |             100 | Akali    | Nocturne | K'Sante      | Lee Sin   | Wukong    | Kalista      |   13.24 | LNG Esports | bot        |
| 10660-10660_game_1 |             200 | Poppy    | Ashe     | Neeko        | Vi        | Jarvan IV | Renata Glasc |   13.24 | Rare Atom   | sup        |
| 10660-10660_game_2 |             100 | Nocturne | Udyr     | Renata Glasc | Nautilus  | Lee Sin   | Neeko        |   13.24 | LNG Esports | mid        |
| 10660-10660_game_2 |             200 | Poppy    | Ashe     | Rumble       | Tristana  | Lucian    | Kalista      |   13.24 | Rare Atom   | bot        |
| 10660-10660_game_3 |             100 | Rell     | Nocturne | Tristana     | Jarvan IV | Rumble    | Neeko        |   13.24 | LNG Esports | mid        |

Still, we can do more. It's possible that individual players favour certain champions. Much like we plucked position from the main dataframe by isolating and then merging it in, we can do the same along the same technical lines with **playername**, adding it to our DataFrame.

| gameid             |   participantid | ban1     | ban2     | ban3         | ban4      | ban5      | pick1        |   patch | teamname    | position   | playername   |
|:-------------------|----------------:|:---------|:---------|:-------------|:----------|:----------|:-------------|--------:|:------------|:-----------|:-------------|
| 10660-10660_game_1 |             100 | Akali    | Nocturne | K'Sante      | Lee Sin   | Wukong    | Kalista      |   13.24 | LNG Esports | bot        | GALA         |
| 10660-10660_game_1 |             200 | Poppy    | Ashe     | Neeko        | Vi        | Jarvan IV | Renata Glasc |   13.24 | Rare Atom   | sup        | Zorah        |
| 10660-10660_game_2 |             100 | Nocturne | Udyr     | Renata Glasc | Nautilus  | Lee Sin   | Neeko        |   13.24 | LNG Esports | mid        | Scout        |
| 10660-10660_game_2 |             200 | Poppy    | Ashe     | Rumble       | Tristana  | Lucian    | Kalista      |   13.24 | Rare Atom   | bot        | Assum        |
| 10660-10660_game_3 |             100 | Rell     | Nocturne | Tristana     | Jarvan IV | Rumble    | Neeko        |   13.24 | LNG Esports | mid        | Scout        |

Almost there, but there's still one glaring issue with our DataFrame. In a match, 10 champions are banned. Right now, those are split across participantid **100** and **200** in sets of 5 apiece, by team. However, we want to remove the 2nd team (id **200**)'s bans and merge them in as new columns, afterwards deleting the 2nd team's row.

We isolate the opponent bans, resulting in a DataFrame which looks like this:

| gameid             | ban6    | ban7     | ban8    | ban9     | ban10     |
|:-------------------|:--------|:---------|:--------|:---------|:----------|
| 10660-10660_game_1 | Poppy   | Ashe     | Neeko   | Vi       | Jarvan IV |
| 10660-10660_game_2 | Poppy   | Ashe     | Rumble  | Tristana | Lucian    |
| 10660-10660_game_3 | Poppy   | Ashe     | LeBlanc | Sejuani  | Vi        |
| 10660-10660_game_4 | Rell    | Nocturne | Ashe    | Azir     | Akali     |
| 10661-10661_game_1 | Kalista | Nocturne | Neeko   | Sejuani  | Poppy     |

Then we merge on gameid, eliminate the now-duplicate rows for the 2nd team (id **200**), and drop NaN values. Our DataFrame for evaluation and training is now ready (broken into 2 rows for legibility, but these are the same DataFrame).

| ban1     | ban2     | ban3         | ban4      | ban5    | pick1   | patch  |
|:---------|:---------|:-------------|:----------|:--------|:--------|-------:|
| Akali    | Nocturne | K'Sante      | Lee Sin   | Wukong  | Kalista | 13.24  |
| Nocturne | Udyr     | Renata Glasc | Nautilus  | Lee Sin | Neeko   | 13.24  |
| Rell     | Nocturne | Tristana     | Jarvan IV | Rumble  | Neeko   | 13.24  |
| Poppy    | LeBlanc  | Neeko        | Sejuani   | Jax     | Rumble  | 13.24  |
| Ashe     | Akali    | LeBlanc      | Vi        | Jax     | Varus   | 13.24  |

| teamname    | position   | playername   | ban6    | ban7     | ban8    | ban9     | ban10     |
|:------------|:-----------|:-------------|:--------|:---------|:--------|:---------|:----------|
| LNG Esports | bot        | GALA         | Poppy   | Ashe     | Neeko   | Vi       | Jarvan IV |
| LNG Esports | mid        | Scout        | Poppy   | Ashe     | Rumble  | Tristana | Lucian    |
| LNG Esports | mid        | Scout        | Poppy   | Ashe     | LeBlanc | Sejuani  | Vi        |
| Rare Atom   | top        | Xiaoxu       | Rell    | Nocturne | Ashe    | Azir     | Akali     |
| JD Gaming   | bot        | Ruler        | Kalista | Nocturne | Neeko   | Sejuani  | Poppy     |

However, for the purposes of our following analyses, we will reflect on the overall, unaltered dataset.

### Univariate Analysis

<iframe
  src="assets/top-30-champions.html"
  frameborder="0"
  width=1000px
  height=500px
></iframe>

In this plot, the top 30 picked champions, we look for any particular outliers that might be useful in predicting pick1 later. It looks like there are a few strong picks at the top that outweigh the rest, Rell, K'Sante, and Nautilus. However, these don't seem to be so drastic that it'd be ideal to treat them differently.

### Bivariate Analysis

<iframe
  src="assets/arcane-banned-bivariate.html"
  frameborder="0"
  width=900px
  height=800px
></iframe>

In this plot, we compare how many times every champion in the set was banned when a certain champion in the set was picked, and repeat this across all entries. The set is non-specific to gameplay, but rather themed after the cast of the popular show Arcane, set in the universe of League of Legends. We see that Vi is the most banned champion, and that there seems to be a relation between Vi and Orianna, where the players of one ban the other.

### Pivot Table

Here, we formed a rather long pivot table of champion bans split across each 'side', blue and red, to get a feel for if that necessarily changed who banned what (since blue picks first in League of Legends, being another way to express team with id **100** vs **200**, as discussed earlier. Only the head is included for the sake of brevity, since there are many champions. I used this to consider whether or not I was going to introduce side as a feature later.

| champion       |   Blue |   Red |
|:---------------|-------:|------:|
| Aatrox         |   1842 |  1644 |
| Ahri           |   1812 |  1794 |
| Akali          |   1950 |  1776 |
| Akshan         |    102 |    78 |
| Alistar        |   2064 |  2742 |
| Ambessa        |     78 |    60 |
| Amumu          |    330 |   258 |
| Anivia         |     96 |    84 |
| Annie          |    144 |   216 |
| Aphelios       |    840 |   660 |

This brings us forward to our assessment of missingness.

## Assessment of Missingness
### NMAR Analysis
A good example of an NMAR missingness column in my dataset is **firsttothreetowers**, which measures the team that was the first to destroy three towers. Since you don't need to destroy three towers to win, or any achieve other data metric in the set, it isn't clear if missing values are missing, or should be imputed with data that neither team destroyed three towers, for a value of **0.0** each in the relevant slots of that column. 

If we had access to the **total number of towers destroyed per team**, we could at least know for sure if we could impute **0.0** for both teams, or make an educated guess on which one reached three first if they are both above that threshold, which would make missingness MAR, dependent on the new **total number of towers destroyed per team** column.

### Missingness Dependency

<iframe
  src="assets/elders-patch-missingness-dependency.html"
  frameborder="0"
  width=800px
  height=600px
></iframe>

Pictured is the Kolmogorov-Smirnov statistical result for our analysis to determine whether or not the missingness of the column 'elders', indicating how many Elder Dragons were killed, was dependent on the column 'patch', which showed what version of the game was being used. We determined to a high degree of confidence that it is not. That means that if we wanted to try to demonstrate that the missingness in 'elders' follows MAR principles, we could not use 'patch' as an example, because it demonstrates no correlation and therefore no dependency.

This brings us, next, to hypothesis testing.

## Hypothesis Testing
When I used to play League of Legends, I played the champion Leona quite a bit. I wanted to investigate if Leona's win rate was reliably above or below 50%. So, I performed a hypothesis test under the following guidelines.

- **H0:** Leona's win rate is not significantly less than 50%.
- **H1:** Leona's win rate is significantly less than 50%.

I ran a binomial hypothesis test, resulting in a win rate of 45.14%, with a p-value of 0.0000038041. **At significance level a = 0.01**, as well as the more permissive a = 0.05, we **reject H0** and say that Leona's win rate is **likely** below 50%. This question is worth investigating for the goal of this study because win rate may be a useful feature to incorporate later for predictive purposes, aggregated across match history. Understanding how common outliers are and understanding what distance from a 50% win rate makes an outlier is useful to know.

This brings us, next, to framing our actual prediction problem.

## Framing a Prediction Problem
### Problem Identification

From the beginning, our question is simple: **can we predict the first champion picked in a game?** We would like to achieve it using only knowledge of which champions were banned, but **do we need more**?

This is a **classification** problem. It is a **multiclass classification** problem, because the question we are answering is not binary. If we were answering whether a champion would be picked or not, that would only have two categories, and therefore be binary. This, however, has a class for every champion. We are predicting **pick1** from the dataset we modified earlier, as that is the stated goal of this study, and it seems like something that should be reasonably predictable based on bans and shifting meta information.

To evaluate our model, we will use **accuracy**. This is because many of the classes are roughly evenly balanced. If we could expect one third or one fourth of all champion entries to belong to a simple champion, then we might consider **F1-score**. However, the champion pool is so widely distributed that accuracy is the best, simplest metric we have.

## Baseline Model
Our Baseline Model uses 10 features, 


