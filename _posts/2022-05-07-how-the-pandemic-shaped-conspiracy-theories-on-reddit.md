---
title: How the Pandemic Shaped Conspiracy Theories on Reddit?
description: Sentiment and topic analysis of r/conspiracy
date: '2022-05-07T21:49:21.503Z'
categories: []
keywords: []
slug: /@nszoni/how-the-pandemic-changed-conspiracy-theories-on-reddit-7739209de425
---

# How the Pandemic Shaped Conspiracy Theories on Reddit?

![](/_posts/img/1__D9i__QX334ctvCdAgNk1mzQ.jpeg)

Sentiment and topic analysis of r/conspiracy

_Codes are available in R and Python_ [_here_](https://github.com/nszoni/reddit_conspiracy_theories)

### Hypothesis

Since the breakout of the pandemic, conspiracy theories have resurfaced and misinformation/fakenews are booming. To confirm that hypothesis, my idea was to use the extensive tools of NLP and compare the posts in the r/conspiracy subreddit before and after COVID.

I expected to see a stronger negative sentiments as well as a shift towards topic connected to society, gender, race, rights, and social media.

### Data Collection

#### Scraping the Reddit API

The traditional way of scraping Reddit is to use the API with the [PRAW](https://praw.readthedocs.io/en/stable/) wrapper. However, the API is limited to 1000 requests per hour and is not able to scrape historical data defined by a window. This is not enough for our purpose while our data would be highly unbalanced and unrepresentative.

#### Overcoming limitations with Pushift API

Luckily there is a service which allows us to scrape historical data from Reddit. The service is called [Pushift API](https://github.com/pushshift/api). Briefly, it let’s you control the scraping window. The python wrapper of the API is called [PSAW](https://psaw.readthedocs.io/en/latest/), but there is also a multi-threaded version called [PMAW](https://github.com/mattpodolak/pmaw) which is more efficient in high volumes close to hundreds of thousands of records.

#### Scraping process

I have used a methodology which blends together the usage of the classical PRAW and PSAW. With PSAW, we only get unique identifiers of submissions from the API bounded by time, which then I feeded into PRAW’s submission endpoint. This way, we can get more features offered by PRAW, but overcome its limitation of scraping historical data.

### Data Ingestion

I have scraped 2500 submissions between 2018 and 2019 for the analysis of conspiracy theories and subreddit activity. For comparison, I also got the same amout of data between 2021 and 2022 which should be a decent baseline for proxying the aftermath of the global pandemic.

### Data Cleaning

#### Feature generation

On Reddit, we you may have seen posts with deleted or removed content either because of the violation of user terms (let’s face it happens often in similar forums) or the author itself decided to delete them. This introduces many records which are not usable for the analysis.

I have decided to glue together the titles and bodies of each post. The limitation of this maneuver is that there are cases where the titles are just the first line of the body text, thus we would end up in information duplication.

```
# replace pattern delete and removed# with missingrc <- rc |>    mutate(body = gsub("\\[deleted\\]|\\[removed\\]","", body))# glue together titles and bodiesrc$text <- paste(rc$title, rc$body, sep = " ")
```

### Text Preprocessing

#### Extend stopwords corpus

The tidytext stopwords corpus is much more extensive than let’s say the nltk stopword corpus in Python (thanks to the multiple of lexicons it uses).

However, I have still arbitrarily extended the corpus with words which does not add to the meaning of the text, but I expect to come up very often based on my experience with nltk.

```
# collect stopwordsdata(stop_words)
```

```
extension <- c("use", "people", "person",    "like", "think", "know", "case", "want",    "mean", "one", "many", "well", "two",    "say", "would", "make", "get", "go",    "thing", "much", "time", "even", "new",    "also", "could")# create dataframe for extensionsextension_df <- data.frame(word = extension, lexicon = rep("custom", length(extension)))
```

```
# unionstop_words <- rbind(stop_words, extension_df)
```

I have created a text processer function which

*   Cleans the text from noises (e.g. username handlers, hyperlinks, non-alphabetic elements, whitespaces, etc.)
*   Tokenizes the text
*   Removes stopwords
*   Lemmatizes the generated tokens
*   Filters out all the words which have less than three characters.

```
text_preprocesser <- function(text) {    # lowercase text    text <- tolower(text)    # remove junk    pattern <- "@[^\\s]+|http\\S+|\\W|\\s+[a-zA-Z]\\s+|\\d+|\\s+"    text <- gsub(pattern, " ", text)    # split to tokens    tokens <- unlist(strsplit(text, "\\s+"))    # filter out stopwords    tokens <- tokens[!(tokens %in% stop_words$word)]    # lemmatize tokens    tokens <- lemmatize_words(tokens)    # filter out tokens with less than 3 characters    tokens <- tokens[length(tokens) >= 3]    # join words back together    preprocessed_text <- paste(tokens, collapse = " ")    return(preprocessed_text)}
```

Applying the text preprocesser, we can see that it did a good job in overall as we managed to filter out stopwords and introduce standardized corpus of lemmatized words. Notice that by removing _in_ from the first post, interpretations can change a lot.

```
rc$cleaned <- lapply(rc$text, text_preprocesser)
```

![](/_posts/img/1__lveS39W__NLIySbztquP5__w.png)

#### Extract emojis

When analyzing social media data, we cannot skip past the fact that there is a trend towards people expressing themselves through emojis and it slowly replaces the means of communication through actual words.

My intention here was to prepare the dataset for analyzing the commonly used emojis across multiple segments (period or sentiment), therefore I have collected all the emojis a submission to a list and added a counter for each using the [emo package](https://github.com/hadley/emo).

```
rc <- rc |>    mutate(emoji = emo::ji_extract_all(text))rc$emoji_count <- sapply(rc$emoji, length)
```

![](/_posts/img/1__QpvcnvwgijjBdQ2__bewh1w.png)

#### TF-IDF

To see what key terms drive each submission by fitting a tf-idf model to the cleaned text.

> The idea of tf-idf is to find the important words for the content of each document by decreasing the weight for commonly used words and increasing the weight for words that are not used very much in a collection or corpus of documents

_Here’s a snippet for that:_

```
rc_tfidf <- rc |>    unnest_tokens(word, cleaned) |>    count(label, word, sort = TRUE) |>    ungroup()total_words <- rc_tfidf |>    group_by(label) |>    summarize(total = sum(n))rc_tfidf <- left_join(rc_tfidf, total_words)
```

```
rc_tfidf <- rc_tfidf |>    bind_tf_idf(word, label, n)tfidf <- rc_tfidf |>    arrange(desc(tf_idf)) |>    mutate(word = factor(word, levels = rev(unique(word)))) |>    group_by(label) |>    top_n(10) |>    ungroup() |>    ggplot(aes(word, tf_idf, fill = label)) +    geom_col(show.legend = FALSE) + labs(x = NULL,    y = "tf-idf") + facet_wrap(~label, scales = "free") +    coord_flip()
```

![](/_posts/img/1__XwdIncLYvtrXHvfOgZf5EQ.png)

It seems that keywords connected to the Kevin Spacey scandal and political conflict (Chinese protest) were the most relevant words of submissions before COVID. After the pandemic, keywords of the vaccination has greatly emerged as the drivers of controversies.

#### Sentiment Analysis

To rank each word by the total sentiment contribute, I have used the AFINN lexicon, and summed up all the scores per label to get the total contribution of a word.

> The AFINN lexicon is a list of English terms manually rated for valence with an integer between -5 (negative) and +5 (positive) by Finn Årup Nielsen between 2009 and 2011.

The below graph suggests that there is a considerable amount of overlap between positive and negative terms pre- and post-COVID.

If I had to pick out certain words which differ, they would be _abuse_ because of all the sexual misconducts before the pandemic and _ban, pay_ and _crisis_ because of the follow-up economic decline after the pandemic and all words related to the sanctions towards Russia.

![](/_posts/img/1__VCmtGniC7IzRxyo81aTSxA.png)

#### Sentiment Categories

We can also leverage other sentiment lexicons, such as NRC, which is great for identifying sentiment categories.

> The NRC Emotion Lexicon is a list of English words and their associations with eight basic emotions (anger, fear, anticipation, trust, surprise, sadness, joy, and disgust) and two sentiments (negative and positive). The annotations were manually done by crowdsourcing.

We can see that the ranking between categories didn’t change much, except for there is relatively more words connected to _trust_. This is mildly connected to the amplified uncertainty after the vaccination myths, where people are questioning theories and fake news are here to stay.

![](/_posts/img/1__u9KyevQaA9kwBOdUxXF__tg.png)

#### Emoji Analysis

Using the list of emojis collected in the text processing section, we can see how the most frequently used emojis have changed over time.

We can see that the top 2 emojis did not change, but there are more negative emojis in after COVID (e.g. skull, clown, suspicious eyes). National flags appear in both terms, from which the Albanian allegedly (had to Google myself) represents Dua Lipa’s tweet backing Albanian nationalism. The Ukrainian flag is there because of the war initiated by Russian aggressors.

![](/_posts/img/1__ABBWJOgqCS24kZd5GZf8KQ.png)

#### Wordclouds

If we check the words associated with sentiments via the bing lexicon, we can see the most common ones in the word clouds below. Conspiracy is still the most used negative word, while Trump is the top positive.

> The bing lexicon categorizes words in a binary fashion into positive and negative categories.

If we look closely at the words which are not overwhelming the rest, we can see that the difference is shifting from abuse, mysterious aliens, impeachment and prison (associated with Harvey Weinstein, UFO findings, Trump, and Jeffrey Epstein, respectively) to virus, bomb, and crisis. As far as positive shift is concerned, there is much less specificity.

![](/_posts/img/1__MSYfP4Te2r4KIK4dwTDf__Q.png)

#### Bigram Analysis

Extending the analysis to multiple words can be a good idea. Thus, I did a tf-idf analysis on bigrams within each submission. Tokens have been strongly connected with Epstein (_die suicide, epstein die, manhattan jail_) and alien-related controversies (_strange creature, creature forest_) while in the post-COVID period, bigrams are more dispersed in terms of topics ranging from vaccination through gender (_conceptual penis_) to cases of sexism (_amp amp_).

```
rc_bigrams <- rc |>    unnest_tokens(bigram, cleaned, token = "ngrams",        n = 2)rc_bigrams_tfidf <- rc_bigrams |>    count(label, bigram) |>    bind_tf_idf(bigram, label, n)
```

![](/_posts/img/1__l3fneOt1vaUz3ng69Y2wmQ.png)

We can even chain together elements of bigrams to see a network of words often mentioned together, creating a whole story. The largest web is around the topics discussing the existence of aliens (bottom-left corner), the Chinese protesters on the Tienanmen square, and the Epstein scandal. These are all from the pre-COVID period, while after the pandemic, there’s been fewer number of persistent stories.

_Snippet:_

```
bigrams_separated <- rc_bigrams |>    separate(bigram, c("word1", "word2"),        sep = " ")# new bigram counts:bigram_counts <- bigrams_separated |>    count(word1, word2, sort = TRUE)bigram_graph <- bigram_counts |>    select(from = word1, to = word2, n = n) |>    filter(n > 20) |>    graph_from_data_frame()a <- grid::arrow(type = "closed", length = unit(0.15,    "inches"))ggraph(bigram_graph, layout = "fr") + geom_edge_link(aes(edge_alpha = n),    show.legend = FALSE, arrow = a, end_cap = circle(0.07,        "inches")) + geom_node_point(color = "lightblue",    size = 5) + geom_node_text(aes(label = name),    vjust = 1, hjust = 1) + labs(title = "Network of bigrams",    subtitle = "Long lineage about Epstein, the aliens, and protests") +    theme_graph()
```

![](/_posts/img/1__gMSE__82d7DYp2klJHwBn3A.png)

#### Topic Modelling (LDA)

Lastly, I wanted to identify topics of discussion before and after the pandemic using Latent Dirichlet Allocation (LDA).

> LDA, short for Latent Dirichlet Allocation is a technique used for topic modelling. Latent means hidden, something that is yet to be found. Dirichlet indicates that the model assumes that the topics in the documents and the words in those topics follow a Dirichlet distribution. Allocation means to giving something, which in this case are topics. LDA assumes that the documents are generated using a statistical generative process, such that each document is a mixture of topics, and each topics are a mixture of words.

#### Pre-COVID

We can see that the topics are ranging from aliens to politics connected with Trump. There are a few topics I can label with mydomain knowledge.

*   Topic 1: [The Epstein case](https://www.bbc.com/news/world-us-canada-48913377)
*   Topic 4: [Alien and UFO discoveries](https://eu.usatoday.com/story/tech/2019/06/11/bizarre-creature-caught-on-security-camera-viral-video/1411177001/)
*   Topic 6: [Google vs. China](https://www.technologyreview.com/2018/12/19/138307/how-google-took-on-china-and-lost/)
*   Topic 7: Social media censorship
*   Topic 8: [Trump’s impeachment](https://en.wikipedia.org/wiki/First_impeachment_of_Donald_Trump)
*   Topic 10: Religion and ethnicity

![](/_posts/img/1__rccKIuDRnY__wmH3WWHMNig.png)

#### Post-COVID

After the pandemic, topics are now ranging from the Ukrainian war through vaccination to social sciences and gender. Other topics are not coherent enough to identify any labels. To name a few topics present here:

*   Topic 1: [Richard Wayne Snell](https://en.wikipedia.org/wiki/Richard_Snell_%28criminal%29) and [Timothy McVeigh](https://en.wikipedia.org/wiki/Timothy_McVeigh) white supremacists and murderers
*   Topic 5: COVID related restrictions and politics (CDC, Canada truck protest)
*   Topic 6: US politics
*   Topic 7: Russian and Ukrainian conflict and a potential nuclear war
*   Topic 17: JFK’s assassination
*   Topic 19: Vaccination
*   Topic 13: Social sciences and gender

![](/_posts/img/1__l4fT5IhNmUv4eo2ALx1Q9g.png)

#### Conclusion

For what is worth, I think we learnt a bunch of new things here and ultimately, my hypothesis held firmly.

Since the pandemic, there is an amplified uncertainty within the community. The relative frequency of words classified in the trust sentiment has increased as well as more and more emojis connected to doubt and negative words have emerged. Topics in the post-COVID period have been less consistent than in the pre-pandemic phase. Many discussions are now about the social science, gender, religion.