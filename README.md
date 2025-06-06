# first-pick-predictor
This is a classification-based prediction model which uses the champion ban list and several other features to predict the first champion picked in a League of Legends game.

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
  width="1000"
  height="900"
  frameborder="0"
  style="max-width: 200%; overflow: auto;">
></iframe>
