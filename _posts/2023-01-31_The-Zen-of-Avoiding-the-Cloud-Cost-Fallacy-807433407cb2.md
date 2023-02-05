---
title: The Zen of Avoiding the Cloud Cost Fallacy
description: Sunk cost in the data space?
date: '2023-01-31T09:38:05.453Z'
categories: []
keywords: []
slug: /@nszoni/the-zen-of-avoiding-the-cloud-cost-fallacy-807433407cb2
---

# The Zen of Avoiding the Cloud Cost Fallacy

[Sunk cost](https://thedecisionlab.com/biases/the-sunk-cost-fallacy) in the data space?

Let’s be honest, 2022 was a good year for data and a lot of companies moved closer to working with data (kinda induced by the pandemic). Also, you probably saw a lot more job openings on LinkedIn to kickstart an [analytics engineering](https://www.getdbt.com/what-is-analytics-engineering/) team. There is still a lot to explore, but more and more companies start to realize how valuable data is in the first place and how dbt shortens the learning curve.

This also means that they probably got slapped with a big chunk of the budget as an investment into digitalization. What do you do when you have money? You SPEND, right?!

![](/_posts/img/1__I2rR0AuLNxQQyOcFiPuiCA.png)

#### Bad signs

Improvements in data analytics in the last decade spoilt us in a way. Extremely cheap storage decoupled from compute completely gave the notion of “scalability” a different meaning and boundaries. On-demand pricing instead of hefty license fees back in the Oracle days gave us flexibility to only pay for what you use. You can always use more resources and increase our capacities to fit our scale. However, [economies of scale](https://www.investopedia.com/terms/e/economiesofscale.asp) are different. You can’t keep up with the exponentially increasing costs as a function of data growth.

> “Sit back, enjoy the ride! We will pay the bills at the end of the month.” — Our inner devil

As the data grows along with the business requirements, you start to see bills getting into a region where it needs to be seriously addressed. Boards are pulled together and Bill asks

> “Hmm why is there a big spike in Snowflake costs, Joe?” — Bill

The team starts to nervously dig into parts of the code where you can cut costs, but oftentimes it seems like they are navigating unknown waters.

![](/_posts/img/1__WU7w1Lc7Y7jfeklsz3PM3g.png)

*   Tools like dbt, Snowflake, BigQuery, etc. push down the cost of operations (which is awesome) but also make us get too comfortable.
*   It’s great to see a plethora of freemium tools, at the end of the day, they keep our warehouses active (not a great combo with Snowflake’s pricing model). Even what’s ‘free’ is only free in its own sense.
*   A data observability tool works usually well if it records the data more frequently, but then it keeps waking up the warehouse\*. Many don’t know that [Snowflake bills for a minimum of 60 seconds](https://docs.snowflake.com/en/user-guide/cost-understanding-compute.html) after firing up a compute resource (i.e. even if your query finishes in 1s, it bills for 60s).

\*does not use warehouse for information schema queries (row count, size, table type, etc) + cached queries.

![](/_posts/img/1__wUtn5FYkNsAX9xIBchvgaQ.png)

*   _Do it well or do it fast._ Codebases are not perfectly polished anymore. Sufficient or good enough code works just fine, it is not worth perfectly optimizing something as long as it’s within our tolerance threshold. This is fine in such a fast-paced environment, as long as you keep track of all the tech debts.
*   [If it ain’t broke, don’t fix it](https://www.merriam-webster.com/dictionary/if%20it%20ain%27t%20broke,%20don%27t%20fix%20it). Spaghetti code leads to indirect costs. (more time for onboarding and implementing a new feature).
*   Vendors are going to charge more. Although business-wise, the [dbt Cloud cost increase](https://www.getdbt.com/blog/dbt-cloud-package-update/) made total sense, announcing it after most of us have agreed on our 2023 budget was a big slap. You should prepare for higher SaaS fees in the future, as the upside is still there to charge for.
*   We drive towards a data space where everyone wants to sell everything (i.e. data observability tools offering cataloguing and profiling). Even things you might even not need.

> _One tool to rule them all, one tool to bring them all — Me_

![](/_posts/img/1__8FbGpkMFPc0XRaLXGP____mw.png)

#### Good Signs

So what else happened in 2022? The global economy got hit seriously and many tech companies overstaffed, and overestimated their productivity, so they started to do mass lay-offs. No, this is not a “good sign” by any means, but it really brought us back to reality to appreciate the cost limiting features of cloud data warehouses.

*   **We started to care.** Snowflake cost estimators are now often requested by our clients, and people in the community, raising awareness of cloud spending.
*   **We understood** that to transparently analyze our costs, we might need to reach deeper into our pockets.
*   **We have more options now**. [Snowflake](https://docs.snowflake.com/en/user-guide/ui-query-profile.html), [BigQuery](https://cloud.google.com/bigquery/docs/query-insights) (finally!) both let us analyze query performance intuitively. This way we can identify where our query falls short (e.g. exploding joins, suboptimal partitions, concurrency issues, so on and so forth). Hopefully, these will also be accessible outside their GUI in the future.
*   **Sturdy barriers**. We can set up a [dogwatch](https://docs.snowflake.com/en/user-guide/resource-monitors.html) on our resources, and BigQuery’s on-demand pricing has a CPU time/data read ratio threshold to keep them profitable. This unconsciously forces us to optimize.

#### What can you do?

If you abide to these laws, you can scale better as a team and have a better ROI to “recover” the sunk costs.

> Btw, Our CTO, Andras has already written a handy [article](https://hiflylabs.com/2022/12/01/using-dbt-with-snowflake-optimization-tips-tricks-part-2/) about cutting costs in Snowflake.

*   **Appreciate all** the great things vendors made available to us. Use [partitioning](https://medium.com/teads-engineering/bigquery-ingestion-time-partitioning-and-partition-copy-with-dbt-cc8a00f373e3) and clustering to facilitate partition pruning. Use well what makes this space modern (e.g. ZCC, table clones, external tables, incremental loading, etc.)
*   **Don’t overdo** partitioning and clustering! Partition on small and very large blocks doesn’t make any sense. Clustering is costly, but works well if you frequently hit the key in the BI tool.
*   **Combine well.** Partitioning and clustering work well together, scan less data per query, and pruning is determined before query start time, embrace them!
*   **Don’t overuse** dbt tests. They are amazing, but test [only what’s needed](https://docs.getdbt.com/blog/grouping-data-tests?hss_channel=lcp-10893210) and test it once (don’t do unnecessary tests again downstream if the logic does not change). Never lose track of tests with warning severity, otherwise, they are redundant.
*   **Understand how** your warehouse costs are generated by starting with the underlying [architecture](https://select.dev/posts/snowflake-architecture) and [query optimizer](https://teej.ghost.io/understanding-the-snowflake-query-optimizer/). The pricing models of Snowflake and BigQuery are different, although both offer flat rate pricing. The former bills on warehouse activity, while the latter on the data processed.
*   BigQuery does a good job balancing performance & price. Storage is inexpensive, and computing feels more flexible. BigQuery excels when query traffic is low. Snowflake doesn’t employ the pay-per-query paradigm, and turning on and off virtual warehouses isn’t quite the same thing.
*   **Choose the right** warehouse, warehouse suspend strategy, table type for your use case.
*   Keep auto suspend at 60s, it’s likely you don’t need more. [Use a set of warehouse](https://select.dev/posts/configuring-snowflake-warehouse-sizes-in-dbt) configurations for differing workloads.
*   **Ask the right questions.** Know the expectations and tolerance of your business stakeholders. Does it bother them if a job runs for a bit long, using a smaller warehouse on Snowflake to save X$? How frequently do you need to see certain data refreshed? Does it make sense to build X times per day? Sometimes less is more.
*   **Evaluate alternatives.** There is fierce competition in this space, and it might be the case that you can find something more suitable for your use-case and [save fortunes](https://medium.com/@datajuls/why-i-moved-my-dbt-workloads-to-github-and-saved-over-65-000-759b37486001). They might need more maintenance (=time=money), but most of these products are now made under the notion of ‘plug and play’.
*   **Tread carefully**. Today, we live in an attention economy, where product usage is likely driven by FOMO and companies are pouring money into their marketing. Trying out new products is always fun, but never lose sight of the gains and losses.

Do we have to care more about optimization ourselves and do our research, or is it fair to expect better query optimization and these advanced tools to identify our bottlenecks?

Either way, my prediction for 2023 is that cloud costs are going to be a counterpoint topic when these vendors go head-to-head, and we are going to see more companies on focusing pushing down the monthly bill.

#### **Shameless plug** 🔌

If cost optimization is a concern for your company, don’t miss our next Snowflake webinar!

On the 8th of February 2023, [Andras Zimmer](https://www.linkedin.com/in/ACoAAAFE5ioBCiRBGtMsWmANM0Vas5wBEdPpXjM?lipi=urn%3Ali%3Apage%3Ad_flagship3_feed%3B175x8KsyS5egStDD8KwT4Q%3D%3D), Head of Analytics Engineering at Hiflylabs is going to walk you through multiple real-life applicable tips & tricks that can help your Snowflake costs plummet.

Book your seat [here](https://us02web.zoom.us/webinar/register/WN_ySTuPCsFR-6UBqLLdiKerw)!

#### Finally, a list of great resources:

*   [https://select.dev/posts/cost-per-query](https://select.dev/posts/cost-per-query)
*   [https://select.dev/posts/introduction-to-snowflake-clustering](https://select.dev/posts/introduction-to-snowflake-clustering)
*   [https://cloud.google.com/bigquery/docs/partitioned-tables](https://cloud.google.com/bigquery/docs/partitioned-tables)
*   [https://hiflylabs.com/2022/12/01/using-dbt-with-snowflake-optimization-tips-tricks-part-2/](https://hiflylabs.com/2022/12/01/using-dbt-with-snowflake-optimization-tips-tricks-part-2/)
*   [https://teej.ghost.io/understanding-the-snowflake-query-optimizer/](https://teej.ghost.io/understanding-the-snowflake-query-optimizer/)
*   [https://teej.ghost.io/understanding-the-snowflake-q](https://teej.ghost.io/understanding-the-snowflake-query-profile/)
*   [https://www.thoughtspot.com/blog/cloud-cost-optimization-4-steps-to-reduce-cloud-costs](https://www.thoughtspot.com/blog/cloud-cost-optimization-4-steps-to-reduce-cloud-costs)