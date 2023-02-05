---
title: Reddit Text Mining with AWS AI Services and R
description: Understanding r/hungary
date: '2021-11-24T18:46:39.534Z'
categories: []
keywords: []
slug: /@nszoni/reddit-text-mining-with-aws-ai-services-and-r-6b055c10725d
---

# Reddit Text Mining with AWS AI Services and R

![](/_posts/img/1__S__u0rYaSnlDI76iHOgZkZw.png)

### Introduction

I’ve always been captivated by the Reddit community, and I frequently find myself looking for solutions and project ideas in the large number of channels, which Redditors refer to as subreddits.

I realize this blogpost’s title is a bit of a departure from my past entries, but there’s a purpose for it. After just recently discovering the [AI Services](https://aws.amazon.com/machine-learning/ai-services/) offered by AWS as a part of my university course, I was assigned to come up with a use-case where these tools can be handy enough.

So, combining my fondness for a primarily text-based social media platform with my university duties, I finally decided to give it a shot and do an **entry-level text analysis of the** [**r/hungary**](https://www.reddit.com/r/hungary/) **subreddit**.

_(Please note that by no means I’m experienced in NLP and this is only a small gig of mine.)_

But why r/hungary, you might wonder? Why not [r/wallstreetbets](https://www.reddit.com/r/wallstreetbets/) or any other internationally known subs? Since I’m based in Budapest, not only this community has pretty much replaced all the Hungarian mainstream media for me, but also been my go-to place after a long workday.

In this tutorial/analysis, I will show you how to get quick-fire analytics and insights from any subreddits of your choice. Also, I will benchmark how AWS Translate and Comprehend perform on slang-powered texts.

**Strap yourselves in!**

> All the artifacts, including the R script behind this blogpost can be found on my [Github repo](https://github.com/nszoni/reddit-sentiments-aws)

### Requirements

#### R

To reproduce the workflow, you need to install the following packages to your R environment.

*   **Data Manipulation and Visualization:** [_tidyverse_](https://www.tidyverse.org)_,_ [_stringr,_](https://www.rdocumentation.org/packages/stringr/versions/1.4.0) [_data.table_](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-intro.html)
*   **Text Mining:** [_tm_](https://cran.r-project.org/web/packages/tm/tm.pdf)_,_ [_tidytext_](https://cran.r-project.org/web/packages/tidytext/vignettes/tidytext.html)_,_ [_SnowballC_](https://cran.r-project.org/web/packages/SnowballC/SnowballC.pdf)
*   **AWS AI from** [**CloudyR**](https://github.com/orgs/cloudyr/repositories)**:** _aws.translate, aws.comprehend_
*   **Accessing Reddit:** [_RedditExtractoR_](https://github.com/ivan-rivera/RedditExtractor)

> _Tip: You can install and load these libraries, with repeating_ install.packages() _and_ library() _commands, or let_ [pacman](https://www.rdocumentation.org/packages/pacman/versions/0.5.1) _deal with the project dependencies in a tidy way._

#### AWS

You also need to set up our AWS credentials to be able to access our cloud resources.

> Note, that the below steps are working under the assumption that you already have an IAM user, if not, [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) you can find how to do so.

1.  Open the IAM console at [https://console.aws.amazon.com/iam/](https://console.aws.amazon.com/iam/).
2.  On the navigation menu, choose **Users**.
3.  Choose your IAM username.
4.  Open the **Security credentials** tab, and then choose **Create access key**.
5.  To see the new access key, choose **Show**. Your credentials should resemble the following examples:

*   Access key ID: `AKIAIOSFODNN7EXAMPLE`
*   Secret access key: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`

To download the key pair, choose **Download .csv file**. Store the .csv file with keys in the same directory as your R script.

Finally, load your credentials into the R session (make sure that your `*accessKeys.csv` is in your working directory):

### Subreddit Processing

#### Reddit API

The next step is to scrape data from the public Reddit API. As a shortcut, I used [**RedditExtractoR**](https://github.com/ivan-rivera/RedditExtractor), which is a minimalistic wrapper for making Reddit API calls in R. In short, the package can extract, user data, subreddit posts, and find URLs to threads based on keywords.

#monthly top entries  
hungary1 <- find\_thread\_urls(subreddit="hungary", sort\_by="top", period = "month")

Peeking into the response data frame, you should see something with the similar structure to this:

![](/_posts/img/1__eVdOZgjDtGwFzQAPf7p1KA.png)

I can already make some preliminary remarks on the data before I jump into processing the data.

1.  Encoding is a bit off as **Hungarian “Ő”s were not parsed correctly**. This is a limitation of our API wrapper, and there is no clear-cut way to remedy that.
2.  It would be reasonable to **glue together the titles and their texts**, to minimize the information loss when mapping text detectors algorithms to a single column of strings.
3.  Subreddit and URL columns are redunndat to my analysis, so let’s just exclude them.
4.  **Posts without body texts should be dropped**, because of ambiguity, and oftentimes titles are not long enough to adhere to the requirements of our analysis.
5.  **Posts having URL links** inside their title/body can cause future problems when processing.
6.  **Too long, or too short texts would make AWS AI Services bail out** due to their character processing limits (max 5,000 characters per request).

Having that in mind, the snippet below should treat all but the first of these issues:

**I started with 1,000 observations and ended up with 172** after narrowing down the dataset to fit our needs. This should be enough to present my use case and respect the budget of my university.

#### Super-function for Processing

Now, I can start detecting the key sentiments and phrases of this subreddit.

For that, I created a **super-function that loops through the rows of my data frame** post-by-post and follows these steps:

1.  **Detects its language and translate** foreign posts to English via _AWS Translate_
2.  **Preprocesses the text** via the _tm_ package (i.e., removes numbers, punctuation, whitespace, capitals, stop words, etc.).
3.  **Detects the sentiment** of the processed texts and **extracts the key phrases** with the highest confidence scores via _AWS Comprehend_

I have also equipped the function with some [logging](https://github.com/daroczig/logger) for debugging purposes, then fed my filtered data into the super-function:

\> hungary\_processed\_test <- subreddit\_processer(hungary5)  
INFO \[2021-11-21 12:20:22\] Preprocessing post No. 1  
INFO \[2021-11-21 12:20:22\] Translating...  
INFO \[2021-11-21 12:20:22\] Detect sentiments...  
INFO \[2021-11-21 12:20:23\] Detect keyphrases...  
INFO \[2021-11-21 12:20:23\] Post No. 1 processed!  
INFO \[2021-11-21 12:20:23\] Preprocessing post No. 2

The whole process **took about 2.2 minutes to complete**, and now I have something much more digestible to do my analytics on.

![](/_posts/img/1__2iyHq7gXN6368LXhR1P4CQ.png)

As you can see, the super-function **did a great job in getting rid of stop words, and all sorts of grammatical objects** which can cause issues in our further data munging. However, I have noticed **inconsistencies in translating heavily informal Hungarian texts** to English as in certain cases, translations weren’t comparable to the original texts or didn’t make sense at all. This is also partly due the encoding issues mentioned before.

#### Keyphrases

To improve our insights about the sentiment of the subreddit, I constructed a **word corpus out of the keyphrases recognized by AWS**. The reason for doing it on keyphrases rather than the processed texts is that I wanted to double down on the filtering of common words which were not part of the out-of-the-box stop words dictionary (e.g. like, think, etc.).

For creating the corpus, I **tokenized the column** of key phrases, counted each word by frequency. After, I **stemmed the list of words** to avoid duplication of words with different suffix and prefixes, and finally added up the frequencies grouped by words to **deduplicate word entries**.

#### Entities

[Entity detection](https://gist.github.com/nszoni/1d7c7efc78ae6f259f8d920002ed76fc) is also part of AWS Comprehend, thereby, I can leverage that to see the **Top 10 entities of the community**. Just as with keyphrases, I had to create a frequency table for every entity appearing in the processed text.

### DataViz

After making sure that everything is in order, I used [ggplot](https://ggplot2.tidyverse.org/index.html) to extract some visual insights from the views I have created in the previous steps.

#### Character Distribution of Top Monthly Posts

The distribution of the number of characters per post follows a **right-skewed** distribution, with the average being approximately **746 characters long**. Such statistics should give me more confidence in the reliability of the text analysis.

![](/_posts/img/1__t0qWd__vYennRQJBaRsImVQ.png)

#### Keywords of the Subreddit

To illustrate the most commonly used phrases in the r/hungary subreddit, I used the stemmed and tokenized key corpus. In my experience, this subreddit is **somewhat politically fuelled**, which reflects why **topics like vaccination, political parties (e.g., _Fidesz_)**, are falling under the most common discussions. In addition to controversial themes, members of the community are usually debating on **family, school, social life and workplace-related** opinions.

![](/_posts/img/1__rJH7chcYlvaY85UNczTaxw.png)

#### Top Entities

Let’s see what were the top entities among the hottest monthly posts.

![](/_posts/img/1__nLdOPfxGoRp7rIH7__myZpg.png)

_Fidesz_ as a political party also ranks in the top entries among generic entities directly associated to the nationality of the subreddit. Due to the nature of Reddit and improper encoding, I can already detect anomalies in word stemming and processing unstructured messages that are not grammatically correct.

#### Weekly Activities in Last Month’s Top

When I look at the weekly distribution of activity in articles sorted by “Last Month’s Top,” I can observe that **Week 42 was relatively a quieter period in terms of comments under top posts**. Meanwhile, the activities on the remaining weeks from last months have been hovering around five thousand comments.

What’s more, **the average weekly number of comments among last month’s top contents was 4,489** which is impressive considering the size of the subreddit.

![](/_posts/img/1__tMXs9NoQl__tgB7W__9eVTSA.png)

Unfortunately, it is not possible to retrieve the number of upvotes on each post from the Reddit API wrapper, which would have made a difference in the definition and standardization of the activity.

#### Weekly Sentiments

Now that I have seen the descriptive statistics about the community, let’s discuss what sentiments drove people towards interaction in the last month. The proportional stacked bar chart below indicates that **positive posts are rarely making it to the top threads.** Also, the **most negative posts were in Week 43 and 44**, **while the least in Week 42**. The negativity in the former-mentioned period could be attributed to the large political upheaval right after the October 26th March and the premiere of a film which have divided supporters of the two political sides.

![](/_posts/img/1__jMg5e7R3JV07agPjasdoJQ.png)

> In charts having multiple colors in the aesthetic, I used ggplot’s [viridis](https://ggplot2.tidyverse.org/reference/scale_viridis.html) scale which is a colorblind-friendly color palette.

#### Top Negative Posts

In the last two subsections, I will dwell on the most active negative and positive posts over the last month by top keyphrases.

Here’s the top 5 posts with the most comments having a negative sentiment assigned to them:

![](/_posts/img/1__kwd__wsBgabZZ3pidwXAfmg.png)

The negative post with the most activity (_“people”_ as its keyword) goes like this:

> How do I find out my colleague that I’m not interested in religion and God-related things? … You don’t keep that respect that I have a different opinion. Is it even possible to get people like that not to make a pledge? …

Unusual wording, but you get the idea. This also reflects the **weaknesses of AWS Translate when handling text full of urban terms**. Also, it’s debatable why religion isn’t the most prominent keyword associated with this passage.

#### Top Positive Posts

As I highlighted before, there’s not been much positivity in recent month’s top posts, apart from asking for literature recommendations and a member sharing his experience about being learning Hungarian as a foreigner for six years now.

![](/_posts/img/1__C4sMA8NtxWeI07L4Dz__tnQ.png)

Again, here’s a short snippet of the top positive post about the literature recommendation:

> Could you recommend a literature whose writer clearly understands science, philosophy of science, and that permeates the book? … I do not even mean that, say, detailed botanical facts arise, but …

Again, _“botanical facts”_ is a very specific keyphrase, but I wouldn’t say that it fully represents the post itself.

### Takeaways

Overally, I have enjoyed working on this blogpost not only because of Reddit but also because I was able to demonstrate a use case for a handful of AWS’s unique services.

In retrospect, all of the components I utilized to analyze r/hungary were simple enough to get some quick-fire insights about the Reddit community. It is undeniable, that there is still room to improve both at the source and processing level, such as the lack of upvote data on posts, further preprocessing, and fine-tuning the extraction of key phrases from the data.

It is also worth investigating how AWS’ sentiment detection works with texts consisting of emojis, how it compares to offerings from other cloud services, and further analysing the performance gap between structured formal texts and texts I have shown you in this blogpost.

> I hope you liked this brief journey with me, and please feel free to leave a remark on your experience with these services!