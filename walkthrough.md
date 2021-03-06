Monte Carlo Simulation in R: Predicting the Snooker World Championships
================

<br>

### Health warning

The code in this repo is intended to demonstrate Monte Carlo simulation
when applied to a sports tournament - I make no promises of
profitability if you used this for betting.

### Some background

The World Championships are the biggest event in the snooker calendar.
The tournament has a pure knockout format. The top 16 players in the
world enter the first round automatically as seeds while a further 16
join them through a series of qualifiers. Unseeded players are randomly
drawn to face seeded players with the paths to the final pre-determined.
There is a good diagram on the [tournament Wikipedia
page](https://en.wikipedia.org/wiki/2022_World_Snooker_Championship).

Ronnie O’Sullivan won his record-equally 7th championship, though the
code could be reused for future tournaments.

Each match consists of a number of frames which varies by round:

-   the Last 32 is best of 19 frames,
-   the Last 16 is best of 25
-   the Quarter Finals are best of 25
-   the Semi Finals are best of 33
-   the Final is best of 35

### Data

Two data files are supplied, `players.rds` and `player-ratings.rds`.

The first, `players.rds` contains the players and their match ups for
the first round of the tournament in the order they were drawn in.

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

    ## here() starts at /Users/neilcurrie/freelance/portfolio/snookermontecarlo

    ## 
    ## Attaching package: 'matrixStats'

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     count

``` r
players <- "{here::here()}/data/players.rds" %>% 
  glue::glue() %>% 
  readr::read_rds()

print(players)
```

    ## # A tibble: 16 × 2
    ##    player1           player2           
    ##    <chr>             <chr>             
    ##  1 Mark Selby        Jamie Jones       
    ##  2 Yan Bingtao       Chris Wakelin     
    ##  3 Barry Hawkins     Jackson Page      
    ##  4 Mark Williams     Michael White     
    ##  5 Kyren Wilson      Ding Junhui       
    ##  6 Stuart Bingham    Lyu Haotian       
    ##  7 Anthony McGill    Liam Highfield    
    ##  8 Judd Trump        Hossein Vafaei    
    ##  9 Neil Robertson    Ashley Hugill     
    ## 10 Jack Lisowski     Matthew Stevens   
    ## 11 Luca Brecel       Noppon Saengkham  
    ## 12 John Higgins      Thepchaiya Un-Nooh
    ## 13 Zhao Xintong      Jamie Clarke      
    ## 14 Shaun Murphy      Stephen Maguire   
    ## 15 Mark Allen        Scott Donaldson   
    ## 16 Ronnie O'Sullivan David Gilbert

The second dataset, `player-ratings.rds` contains two columns, one with
the name of the player and one with their rating. The rating is roughly
their win percentage but adjusted for quality of opposition. There are a
variety of ways you could calculate this rating or approaches you could
take which I won’t go into here but this slide deck [Modelling Outcomes
of Professional Snooker
Matches](http://www2.stat-athens.aueb.gr/~jbn/conferences/MathSport_presentations/TRACK%20B/B3%20-%20Sports%20Modelling%20and%20Prediction/Collingwood_Snooker.pdf)
is a great place to start.

``` r
player_ratings <- "{here()}/data/player-ratings.rds" %>% glue() %>% read_rds()

player_ratings %>%
  dplyr::arrange(dplyr::desc(rating)) %>%
  print()
```

    ## # A tibble: 145 × 2
    ##    player            rating
    ##    <chr>              <dbl>
    ##  1 Judd Trump         0.674
    ##  2 Neil Robertson     0.653
    ##  3 Ronnie O'Sullivan  0.645
    ##  4 John Higgins       0.642
    ##  5 Mark Selby         0.623
    ##  6 Kyren Wilson       0.616
    ##  7 Mark Allen         0.613
    ##  8 Yan Bingtao        0.613
    ##  9 Mark Williams      0.611
    ## 10 Barry Hawkins      0.597
    ## # … with 135 more rows

### Predicting a frame

Firstly we define a function for predicting the outcome of a frame. The
first argument `match_rating` is the rating of player 1 minus the rating
of player 2. A positive match rating indicates that player 1 is better
while a negative match rating indicates the opposite.

The second argument scaling_para scales the probability prediction. I
found a good value for this parameter from back testing on frame data to
be 0.7. You could do this using grid search or R’s built in optim or
optimise functions.

``` r
calc_frame_prob <- function (match_rating, scaling_para = 0.7) {
  
  0.5 + scaling_para * match_rating
  
}
```

### Predicting a match

This modelling approach assumes that the probability of a player winning
a frame is the same throughout the match. By making this assumption we
can analytically calculate match probabilities using the binomial
distribution. More information about this can be found at [Statistics
How
To](https://www.statisticshowto.com/probability-and-statistics/binomial-theorem/binomial-distribution-formula/)
.

For each possible score, we will calculate the probability of that score
occurring, then sum together the probabilities for each possible score
to get the match probabilities for each player.

There is a more in depth description of this process in the [Snooker
Predictions site](https://snooker-predictions.com/algorithm.html) though
the probability of winning a frame in this case is found using an ELO
model.

We require functions to find possible scores given a best of, calculate
the number of paths to those scores and to bring it all together to
calculate the probability of winning a match.

``` r
build_possible_scores <- function (best_of) {

  first_to <- ceiling(best_of / 2)
  
  scores <- c(rep(first_to, first_to), 
              0:(first_to - 1), 
              0:(first_to - 1), 
              rep(first_to, first_to))
  
  score_matrix <- matrix(scores, ncol = 2, 
                         dimnames = list(NULL, c("score1", "score2")))

}

calc_num_paths_to_score <- function (possible_scores) {

  x <- matrixStats::rowMaxs(possible_scores) - 1
  y <- matrixStats::rowMins(possible_scores)

  factorial(x + y) / (factorial(x) * factorial(y))

}
```

The following will calculate the probability of winning a match for
player 1 given that players probability of winning a frame and the best
of. Matches played over more frames will favour better players.

``` r
calc_match_prob <- function (prob_frame, best_of) {

  possible_scores <- build_possible_scores(best_of)
  paths_to_score <- calc_num_paths_to_score(possible_scores)

  prob_winning_scores <- possible_scores %>%
    tibble::as_tibble() %>%
    dplyr::mutate(permutations = paths_to_score,
                  prob_frame,
                  prob_score = permutations * (prob_frame ^ score1) * 
                               ((1 - prob_frame)) ^ score2) %>%
    filter(score1 > score2)
  
  prob_match <- sum(prob_winning_scores$prob_score)
  
}
```

### Simulating the tournament

We now have the ability to predict a match. Now comes the simulation
part. We will construct a list which contains the structure of the
tournament. Each list element is a data frame / tibble corresponding to
a round. From round 2 onward we have columns player1_prev_match and
player2_prev_match which allow us to allocate winning players to the
correct match in the next round.

``` r
build_tournament <- function (num_players) {

  # Initialise the tournament list
  
  matches_r1 <- tibble(.round = "r1",
                       match_num = 1:(num_players / 2),
                       player1 = NA_character_,
                       player2 = NA_character_)

  matches_r <- matches_r1

  num_rounds <-  log(num_players, base = 2)

  store_rounds <- vector("list", num_rounds)

  store_rounds[[1]] <- matches_r1

  # Loop through each subsequent round
  
  for (i in 2:num_rounds) {

    prev_match_nums <- matches_r$match_num
    prev_round_max_match_num <- max(prev_match_nums)
    prev_round_num_matches <- nrow(matches_r)
    curr_round_num_matches <- prev_round_num_matches / 2

    matches_ri <- tibble(
      .round = paste0("r", i),
      match_num = (1 + prev_round_max_match_num):(prev_round_max_match_num + curr_round_num_matches),
      player1_prev_match = NA_real_,
      player2_prev_match = NA_real_,
      player1 = NA_character_,
      player2 = NA_character_
      )

    # Populate the prev_match columns
    
    prev_match_num_counter <- 0

    for (j in 1:nrow(matches_ri)) {

      for (k in 1:2) {

        prev_match_num_counter <- prev_match_num_counter + 1

        matches_ri[j, 2 + k] <- prev_match_nums[prev_match_num_counter]

      }

    }

    store_rounds[[i]] <- matches_ri
    matches_r <- matches_ri

  }

  return(store_rounds)

}
```

And now we will write a function to simulate a tournament. For each
match we will use the functions we previous wrote to calculate match
probabilities. We will generate a random number to decide who won the
match and put them into the next round, recording what stage each player
got to. The `results` argument is a matrix for storing what stage the
player got to while `best_ofs` corresponds to the number of frames
played for each round of the tournament.

``` r
simulate_tournament <- function(tournament, player_ratings, players, results, 
                                best_ofs, scaling_para = 0.7) {
  
  players <- c(tournament[[1]]$player1, tournament[[1]]$player2)
  num_players <- length(players)
  num_rounds <- length(tournament)
  
  for (i in 1:num_rounds) {

    tournament[[i]] <- tournament[[i]] %>%
      dplyr::left_join(player_ratings, c("player1" = "player")) %>%
      dplyr::rename(player1_rating = rating) %>%
      left_join(player_ratings, c("player2" = "player")) %>%
      rename(player2_rating = rating) %>%
      dplyr::mutate(
        match_rating = player1_rating - player2_rating,
        player1_frame_prob = calc_frame_prob(match_rating),
        player1_match_prob = purrr::map_dbl(player1_frame_prob, 
                                            .f = calc_match_prob,
                                            best_of = best_ofs[i]),
        rand = runif(n()),
        winner = if_else(rand < player1_match_prob, player1, player2)
        )
    
    winner_indicator <- rep(0, length(players))
    winner_indicator[players %in% tournament[[i]]$winner] <- 1

    results[,i] <- results[,i] + winner_indicator

    if (i != num_rounds) { # Not required on last round

      tournament[[i + 1]][, "player1"] <- tournament[[i]] %>%
        filter(match_num %in% tournament[[i + 1]]$player1_prev_match) %>%
        dplyr::pull(winner)

      tournament[[i + 1]][, "player2"] <- tournament[[i]] %>%
        filter(match_num %in% tournament[[i + 1]]$player2_prev_match) %>%
        pull(winner)

    }

  }

  return(results)

}

tournament <- build_tournament(32)

player_names <- c(players$player1, players$player2)

results <- matrix(0, nrow = 32, ncol = length(tournament))
rownames(results) <- player_names
colnames(results) <- paste0("r", 1:length(tournament))

print(results[1:5,])
```

    ##               r1 r2 r3 r4 r5
    ## Mark Selby     0  0  0  0  0
    ## Yan Bingtao    0  0  0  0  0
    ## Barry Hawkins  0  0  0  0  0
    ## Mark Williams  0  0  0  0  0
    ## Kyren Wilson   0  0  0  0  0

We now have all our functions in place and are ready to start
simulating. I have set a seed for the random number generator - you
should get the same results for this simulation. We can see that in this
simulation Ronnie O’Sullivan won - there is a 1 in each column in his
row. Further inspection shows Barry Hawkins was runner up - he has a 1
in each column of his row except the last one.

``` r
best_ofs <- c(19, 25, 25, 33, 35)

tournament[[1]]$player1 <- players$player1
tournament[[1]]$player2 <- players$player2

set.seed(101)

sim1 <- simulate_tournament(tournament, player_ratings, players, results, 
                         best_ofs, scaling_para = 0.7)

print(sim1)
```

    ##                    r1 r2 r3 r4 r5
    ## Mark Selby          1  0  0  0  0
    ## Yan Bingtao         1  1  0  0  0
    ## Barry Hawkins       1  1  1  1  0
    ## Mark Williams       1  0  0  0  0
    ## Kyren Wilson        1  1  0  0  0
    ## Stuart Bingham      1  0  0  0  0
    ## Anthony McGill      1  1  1  0  0
    ## Judd Trump          1  0  0  0  0
    ## Neil Robertson      1  1  1  0  0
    ## Jack Lisowski       1  0  0  0  0
    ## Luca Brecel         0  0  0  0  0
    ## John Higgins        1  1  0  0  0
    ## Zhao Xintong        1  1  0  0  0
    ## Shaun Murphy        0  0  0  0  0
    ## Mark Allen          1  0  0  0  0
    ## Ronnie O'Sullivan   1  1  1  1  1
    ## Jamie Jones         0  0  0  0  0
    ## Chris Wakelin       0  0  0  0  0
    ## Jackson Page        0  0  0  0  0
    ## Michael White       0  0  0  0  0
    ## Ding Junhui         0  0  0  0  0
    ## Lyu Haotian         0  0  0  0  0
    ## Liam Highfield      0  0  0  0  0
    ## Hossein Vafaei      0  0  0  0  0
    ## Ashley Hugill       0  0  0  0  0
    ## Matthew Stevens     0  0  0  0  0
    ## Noppon Saengkham    1  0  0  0  0
    ## Thepchaiya Un-Nooh  0  0  0  0  0
    ## Jamie Clarke        0  0  0  0  0
    ## Stephen Maguire     1  0  0  0  0
    ## Scott Donaldson     0  0  0  0  0
    ## David Gilbert       0  0  0  0  0

That’s just one simulation though. The power of this is that you can do
it over and over again, many thousands of times, and the law of large
numbers tells us the results will converge to the true probabilities. I
like using the tictoc package for timing code and outputting where we
are at when a long looping procedure is under way. Below I have set
`num_sim` to be 10 but you would want to make this number much bigger,
say 100k, which will take you a few hours.

``` r
num_sims <- 10

results_original <- results

tictoc::tic()

for (i in 1:num_sims) {
  
  results <- results + simulate_tournament(tournament, 
                                           player_ratings, 
                                           players, 
                                           results_original, 
                                           best_ofs, 
                                           scaling_para = 0.7)
  
  
  if (i %% 1000 == 0) { # prints an update every 1000 simulations
    
    tictoc::toc()
    print(i)
    tic()
    
  }
  
  
}

results_prob <- results %>%
  magrittr::divide_by(num_sims) %>%
  as_tibble() %>%
  mutate(player = rownames(results), .before = r1) %>%
  arrange(desc(r5))

results_odds <- results_prob %>% mutate(across(r1:r5, ~ 1 / .))

print(results)
```

    ##                    r1 r2 r3 r4 r5
    ## Mark Selby         10  6  4  2  1
    ## Yan Bingtao         6  3  1  1  0
    ## Barry Hawkins       7  4  1  1  1
    ## Mark Williams      10  5  3  2  1
    ## Kyren Wilson        6  2  1  0  0
    ## Stuart Bingham      7  5  3  2  1
    ## Anthony McGill      7  1  1  0  0
    ## Judd Trump          8  6  2  2  2
    ## Neil Robertson      7  4  4  4  2
    ## Jack Lisowski       5  5  2  0  0
    ## Luca Brecel         7  2  0  0  0
    ## John Higgins        8  6  4  2  1
    ## Zhao Xintong        6  3  1  1  0
    ## Shaun Murphy        7  5  3  0  0
    ## Mark Allen          9  2  1  0  0
    ## Ronnie O'Sullivan   6  5  4  3  1
    ## Jamie Jones         0  0  0  0  0
    ## Chris Wakelin       4  1  1  0  0
    ## Jackson Page        3  1  0  0  0
    ## Michael White       0  0  0  0  0
    ## Ding Junhui         4  2  2  0  0
    ## Lyu Haotian         3  1  0  0  0
    ## Liam Highfield      3  1  0  0  0
    ## Hossein Vafaei      2  2  1  0  0
    ## Ashley Hugill       3  0  0  0  0
    ## Matthew Stevens     5  1  0  0  0
    ## Noppon Saengkham    3  1  0  0  0
    ## Thepchaiya Un-Nooh  2  1  0  0  0
    ## Jamie Clarke        4  2  1  0  0
    ## Stephen Maguire     3  0  0  0  0
    ## Scott Donaldson     1  0  0  0  0
    ## David Gilbert       4  3  0  0  0

``` r
print(results_prob)
```

    ## # A tibble: 32 × 6
    ##    player               r1    r2    r3    r4    r5
    ##    <chr>             <dbl> <dbl> <dbl> <dbl> <dbl>
    ##  1 Judd Trump          0.8   0.6   0.2   0.2   0.2
    ##  2 Neil Robertson      0.7   0.4   0.4   0.4   0.2
    ##  3 Mark Selby          1     0.6   0.4   0.2   0.1
    ##  4 Barry Hawkins       0.7   0.4   0.1   0.1   0.1
    ##  5 Mark Williams       1     0.5   0.3   0.2   0.1
    ##  6 Stuart Bingham      0.7   0.5   0.3   0.2   0.1
    ##  7 John Higgins        0.8   0.6   0.4   0.2   0.1
    ##  8 Ronnie O'Sullivan   0.6   0.5   0.4   0.3   0.1
    ##  9 Yan Bingtao         0.6   0.3   0.1   0.1   0  
    ## 10 Kyren Wilson        0.6   0.2   0.1   0     0  
    ## # … with 22 more rows

``` r
print(results_odds)
```

    ## # A tibble: 32 × 6
    ##    player               r1    r2    r3     r4    r5
    ##    <chr>             <dbl> <dbl> <dbl>  <dbl> <dbl>
    ##  1 Judd Trump         1.25  1.67  5      5        5
    ##  2 Neil Robertson     1.43  2.5   2.5    2.5      5
    ##  3 Mark Selby         1     1.67  2.5    5       10
    ##  4 Barry Hawkins      1.43  2.5  10     10       10
    ##  5 Mark Williams      1     2     3.33   5       10
    ##  6 Stuart Bingham     1.43  2     3.33   5       10
    ##  7 John Higgins       1.25  1.67  2.5    5       10
    ##  8 Ronnie O'Sullivan  1.67  2     2.5    3.33    10
    ##  9 Yan Bingtao        1.67  3.33 10     10      Inf
    ## 10 Kyren Wilson       1.67  5    10    Inf      Inf
    ## # … with 22 more rows

## Improvements

### 1. Speed

There are a host of speed improvements you could make to this. One
obvious one is the calc_match_prob function which we map over for each
match. Each time the function derives each potential score and the
number of paths to that score to sum over. There are multiple matches in
most rounds so you technically could just do this once per round. Doing
more things with vectors and matrices is likely to be quicker.

### 2. Updating ratings

One thing that I didn’t highlight but you probably picked up on is that
the ratings are static. In reality, a players rating would improve. I
can imagine this would take some speed out of simulations.

### 3. Ratings

I have’t gone into depth about the ratings. There are plenty of sources
of data out there like [Snooker.org](www.snooker.org) or
[Cuetracker](https://cuetracker.net).

### 4. Different stages

You could adapt this to focus on different stages or do simulations part
way through the tournament.
