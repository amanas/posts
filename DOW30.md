Introduction
------------

This article shows how simple it is to generate an **animated graphic**
with the programming language [R](https://www.r-project.org/) and the
library [gganimate](https://gganimate.com).

To take advantage of this work with real data, **we will use economic
data** of the companies of the [Dow Jones Industrial Average (DOW 30
index)](https://en.wikipedia.org/wiki/Dow_Jones_Industrial_Average).

The evolution over the last few months of the value of these companies
is an ideal practical case for this type of animation.

On the other hand, from an economic point of view, sharing this
information can also be of great interest, regardless of the tool with
which it was generated.

Market Capitalization vs. Equity
--------------------------------

**Estimating the net worth or real wealth of a company is a complex
activity.**

In cases where the company is listed on a market, it is not as simple as
multiplying the shares it has by the price of each share. In fact, the
market price of a company’s shares seems to relate more to the
perception that buyers have than to the real value.

For this reason, sometimes it is said that a company is expensive or
cheap in a market (or even that the market is expensive or cheap). It
means that the price of the company (or the market as a whole) is above
or below its real price.

To have both visions, that of market value and that of real value (or
something as similar as possible), we can go to the following two
metrics. According to \[investopedia\]
(<a href="https://www.investopedia.com/" class="uri">https://www.investopedia.com/</a>):

-   [Market
    Capitalization](https://www.investopedia.com/terms/m/marketcapitalization.asp):
    refers to the total dollar market value of a company’s outstanding
    shares of stock. Commonly referred to as “market cap,” it is
    calculated by multiplying the total number of a company’s
    outstanding shares by the current market price of one share.

-   [Enterprise
    Value](https://www.investopedia.com/terms/e/enterprisevalue.asp): is
    a measure of a company’s total value, often used as a more
    comprehensive alternative to equity market capitalization. EV
    includes in its calculation the market capitalization of a company
    but also short-term and long-term debt as well as any cash on the
    company’s balance sheet. Enterprise value is a popular metric used
    to value a company for a potential takeover.

From the above definitions, it seems that it may be interesting to
analyze the evolution over time of both metrics for **DOW 30**
companies.

Data provider
-------------

However, once we know the metrics we want to analyze, **the really
difficult part is finding a reliable and affordable finantial data
source**.

Fortunately, [Tiingo](https://api.tiingo.com/documentation/fundamentals)
makes it possible to access this information for free, precisely for all
the DOW30 companies. Hopefully, in the future Tiingo will offer this
information for a greater number of companies. By the way, the Tiingo
API is one of the simplest and most friendly in its category; Not all
the vendors listed offer such a comprehensive data API and transparent
billing mode.

So first, we can define a function to get the needed **fundamental
data**. As you can see, you will need your own **Tiingo API token**,
which can be easily obtained at [Tiingo](https://tiingo.com). Notice
that we memoise this function to avoid making extra calls to the API.

``` r
tiingo.fundamentals.daily <- function(symbol) {
  tryCatch(
    "https://api.tiingo.com/tiingo/fundamentals/%s/daily" %>%
      sprintf(symbol) %>%
      GET(query = list(token = Sys.getenv("TIINGO_API_TOKEN"))) %>%
      content %>%
      rbindlist(fill = T, use.names = T) %>%
      as_tibble %>%
      add_column(symbol = symbol, .before = 1),
    error = function(e) {
      NULL
    }
  )
}
tiingo.fundamentals.daily.mem <- memoise(tiingo.fundamentals.daily)
```

Data munging
------------

We need the DOW30 list of symbols (companies) to download the necessary
data from Tiingo. We can obtain them using the
[tidyquant](https://cran.r-project.org/web/packages/tidyquant/index.html)
library, which also has other interesting features that go beyond this
article.

``` r
dow30 <- tq_index("DOW")

fundamental.data <- dow30 %>% 
  pull(symbol) %>% 
  lapply(tiingo.fundamentals.daily.mem) %>%
  rbindlist(fill = T, use.names = T) %>% 
  left_join(dow30) %>% 
  select(date,symbol,company,marketCap,enterpriseVal)
```

Since the chart legend has limited space, we want to shorten the company
name. And yes, we are only interested in 2020 data.

``` r
anim.data <- fundamental.data %>% 
  mutate(date = as.Date(date)) %>% 
  filter('2020-01-01' <= date) %>% 
  filter(date <= '2020-12-31') %>% 
  mutate(company = paste0(symbol,": ",company)) %>% 
  mutate(company = ifelse(nchar(company)<=20,
                          substr(company,1,20),
                          paste0(substr(company,1,20),'...'))) %>% 
  as_tibble
```

Now, the values don’t change every day (think weekends, holidays, etc.).
So we’re going **to smooth out those gaps** to generate a smoother
animation:

``` r
dates     <- min(anim.data$date) %>% 
  seq.Date(max(anim.data$date), by="day") %>% 
  tibble(date = .)
symbols   <- anim.data %>% select(symbol) %>% distinct
anim.grid <- merge(dates, symbols)

anim.data <- anim.grid %>% 
  left_join(anim.data) %>% 
  arrange(symbol,date) %>% 
  fill(company) %>% 
  na_interpolation
```

Next, we are not so interested in the absolute numbers, but we are
interested in knowing which company is growing and which is not. So we
need the **percentage of change from January 1, 2020**.

``` r
anim.data <- anim.data %>% 
  arrange(date) %>% 
  group_by(symbol) %>% 
  mutate(firstMCap = first(marketCap),
         firstEVal = first(enterpriseVal),
         diffMCap  = (marketCap - firstMCap) / firstMCap,
         diffEVal  = (enterpriseVal - firstEVal) / firstEVal) 
```

The animation
-------------

So finally we are ready to create our animation.

First by defining it:

``` r
top.symbols <- anim.data %>% 
  select(symbol, firstEVal) %>% 
  distinct %>% 
  arrange(desc(firstEVal)) %>% 
  head(6) %>% 
  pull(symbol)

animation <- anim.data %>% 
  ggplot(aes(x=diffMCap, y=diffEVal, size=enterpriseVal, color=company, frame = date)) +
  geom_point(alpha = 0.5) +
  geom_label(aes(label=ifelse(symbol %in% top.symbols, symbol,
                       ifelse(0.3 < abs(diffMCap),     symbol,
                       ifelse(0.3 < abs(diffEVal),     symbol, NA)))),
             show.legend=FALSE, size=3, vjust = "inward", hjust = "inward") +
  scale_size_area(max_size = 16) +
  scale_x_continuous(labels = function(x) paste0(x*100, "%")) +
  scale_y_continuous(labels = function(y) paste0(y*100, "%")) +
  guides(size  = "none",
         color = guide_legend(title="Company", ncol=1)) +
  ggtitle("DOW30 2020: Jan 1 vs. {format(frame_time, format='%b %d')}") +
  labs(x = "Market Capitalization (% difference)", 
       y = "Enterprise Value (% difference)",
       caption = c("Author: Andrés Mañas","Data provided by Tiingo")) +
  theme(plot.caption = element_text(hjust=c(0,1)))+ 
  transition_time(date) +
  enter_fade() +
  exit_fade()
```

Second, by creating and saving it. As you can see, we want the entire
movie to be 60 seconds long and have 24 frames per second.

``` r
fps=24
duration=60
animation %>% animate(fps=fps, duration=duration, nframes=fps*duration,
                      end_pause=fps*10, rewind=F,
                      height=600, width=800)
anim_save("files/DOW30.gif")
```

And finally, we get a visual representation of the evolution of the
**Market Capitalization** and the **Enterprise Value** for all the DOW
30 companies from the beginning of 2020 to date.

![](files/DOW30.gif)

Conclusion
----------

Economics or finance is not my specialty at all. So this is not about
economics, even though the result could be interesting in those terms.

This is an article on how to make good animations with R and perhaps a
reading that allows you to discover tools for better and more effective
communication. I hope it has been useful to you.

If you want to review the full code, just visit the
[github](https://github.com/amanas/posts/blob/master/DOW30.Rmd)
repository.

If you want more like this, leave your suggestions in the comments.

Thanks for reading!
