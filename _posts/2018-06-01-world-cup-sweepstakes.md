---
layout: post
title:  Organising your Euro 2021 sweepstakes using R
date: "2018-06-01 11:00:00"
image: /figure/source/world-cup-sweepstakes/sweepstakes-og.jpg
share-img: http://www.nathancunn.com/figure/source/world-cup-sweepstakes/sweepstakes-og.jpg
---

_Updated June 3 2021_

Like many others are doing at this time, I'm sure, my friends and I are trying to coordinate a sweepstakes for the upcoming Euro 2021. For anyone unaware, the general format of a Euros sweepstakes is:
24 people (or some factor, thereof) pledge an equal sum of money; each draws a competing nation
 from a hat; the person who draws the winning team takes all the cash. Pretty simple.
 Of course, social distancing and work from home guidelines arising due to Covid make this a little more complicated to organise among work colleagues. 
 In my case, the friends I'm organising with also live in different countries, so short of some sort of Sisterhood of the Travelling Pants style arrangement, drawing teams from a hat is out of the question.

We could have arranged for one trustworthy person to draw a team for everyone from a hat. But, you might discover you trust that person much less if they end up with, say, Germany than if they draw themselves Finland. Another suggestion floated was to do a random sort of names and countries in Microsoft Excel, but this falls victim to the same
issue as before in that the person in charge can do the draw over and over until they get a team they're happy with. And even if they don't, you'll never satisfy everyone that the draw was done fairly, with people's perception of fairness positively correlated to how good their chosen team are.

So, we need to draw teams in a way which we all trust none of us could possibly hold any influence over. Before explaining how I did this in R, I need to give a very brief introduction to pseudo-random numbers.

When you ask your computer for random numbers, what you get aren't strictly
random. For general purposes they're as good as random, but they are actually deterministically generated using an algorithm. Consider generating two sets of five normally-distributed numbers in R; knowing the
first group won't help you figure out what the next will be:
```r
rnorm(5)
[1] -0.47719270 -0.99838644 -0.77625389  0.06445882  0.95949406
rnorm(5)
[1]  0.5060559 -0.5747400 -0.5466319 -0.5644520 -0.8900378
```

 However, these numbers *are* knowable in advance. These are what are referred to as pseudo-random numbers, and their deterministic nature is actually a benefit as opposed to a hindrance. Say you're performing some analysis which has some random aspect to it,
 having an algorithmically generated list of numbers means your results can be reproduced at a later date.

 In order to reproduce a sequence of random numbers, though, you'll need to know what's known as the random seed used to generate those numbers.
 The random seed is the starting point that determines the full chain of random numbers generated. A simple way to think of this is to imagine that your computer
 has a book full of random numbers (these do actually exist [and the reviews are hilarious](https://www.amazon.co.uk/Million-Random-Digits-Normal-Deviates/dp/0833030477)). When you ask for random numbers, the computer opens the book and lists off numbers as needed. The random seed, then, is the page in this book that your computer reads the numbers off. No matter how many times you tell your computer to go to that page, the set of random numbers it returns will always be the same.
 We can specify a random seed in R using the `set.seed()` function, and we see that if we generate a set of random numbers as before,
 this time the results do not change.

```r
set.seed(1234)
rnorm(5)
`[1] -1.2070657  0.2774292  1.0844412 -2.3456977  0.4291247`
set.seed(1234)
rnorm(5)
`[1] -1.2070657  0.2774292  1.0844412 -2.3456977  0.4291247`
```

Now, back to the sweepstakes. Doing the draw is just randomly ordering the countries and matching these to the names. First off, we have to input all the participants and countries into vectors in R. Note that the resulting
pairings will depend on the order that these are input, as such I decided to sort them alphabetically, to avoid any suggestion that the order of input benefits me.

``` r
names <- sprintf("person %d", 1:24)
countries <- sort(c("Italy", "Switzerland", "Turkey", "Wales",
                        "Belgium", "Russia", "Denmark", "Finland",
                        "Ukraine", "Netherlands", "Austria", "North Macedonia",
                        "England", "Croatia", "Czech Republic", "Scotland",
                        "Spain", "Poland", "Sweden", "Slovakia", 
                        "Germany", "France", "Portugal", "Hungary"))

```

So that all of us get the same draw, we'll all need to set the same random seed before drawing countries. But, in order to allay any fears of a fix, we need to specify the random seed using an external source of randomness upon which none of us have control. On top of this, the chosen seed should have enough potential variation to account for
a large number of possibilities---if, for example, you did something like take the average of the day everyone was born, the seed can only take 31 different values (the seed is a whole number), meaning only 31 different draws are possible---and it's likely that none of those possibilities include you getting Germany.

 In my case, I chose the volume traded on the FTSE100 for June 9th, having agreed on this ahead of time. None of us could know this in advance, or have any meaningful influence on it. And really, if we could meaningfully influence the FTSE then we'd have more lucrative uses for that than cheating at a sweepstakes. Furthermore, the range of possible values is large enough to allow for many possibilities to be covered---as there are 32! ways of doing this draw, and the range of likely trading volumes will be considerably less than this we won't cover *every* possibility, but we should cover enough to keep everyone happy.
  We can get the FTSE data using the `quantmod` package in R:

``` r
library(lubridate)
library(quantmod)
draw_date <- "2018-05-31"
getSymbols("^FTSE", from = draw_date - days(1))
seed <- FTSE[draw_date - days(1)]$FTSE.Volume
set.seed(seed)
tibble(Name = names, Country = sample(countries))
```

Now we all can run this code separately and get the same draw and trust that none of us could have influenced the results in our favour. If your friends aren't familiar with R you could do as I did and set up a Shiny App to share with them:

``` r
library(shiny)
# Updated for EURO 2021
ui <- fluidPage(titlePanel("Euro 2021 Sweepstakes"),
                fluidRow(tableOutput('draw')),
                fluidRow(textOutput('text'))
)

server <- function(input, output) {
    library(lubridate)
    library(quantmod)
    library(tidyverse)
    names <- sprintf("person %d", 1:24)
    countries <- sort(c("Italy", "Switzerland", "Turkey", "Wales",
                        "Belgium", "Russia", "Denmark", "Finland",
                        "Ukraine", "Netherlands", "Austria", "North Macedonia",
                        "England", "Croatia", "Czech Republic", "Scotland",
                        "Spain", "Poland", "Sweden", "Slovakia", 
                        "Germany", "France", "Portugal", "Hungary"))
    draw_date <- as_date("2021-06-03")
    if(today() >= draw_date) {
        quantmod::getSymbols("^FTSE", from = draw_date - days(1))
        seed <- FTSE[draw_date - days(1)]$FTSE.Volume
        msg <- paste0("Random seed used: ", seed, " from FTSE daily volume on ", draw_date - days(1))
    } else {
        seed <- round(runif(1) * 10e6)
        msg <- paste0("Random seed used: ", seed, " (selected at random; draw will take place on ", draw_date)
    }
    set.seed(seed)
    sweepstakes <- tibble(Name = names,
                          Country = sample(countries))
    output$draw <- renderTable({
        sweepstakes
    })
    output$text <- renderText(msg)
}


# Run the application
shinyApp(ui = ui, server = server)
```
