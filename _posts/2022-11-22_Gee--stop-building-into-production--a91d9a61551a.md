---
title: 'Gee, stop building into production!'
description: How I Learned to Stop Worrying and Love the Deployment
date: '2022-11-22T11:51:13.122Z'
categories: []
keywords: []
slug: /@nszoni/gee-stop-building-into-production-a91d9a61551a
---

_How I Learned to Stop Worrying and Love the Deployment_

![](/_posts/img/1__qXpM8o49P6kSnugDycbn__A.png)

### 🌀Why all the fuss?

Time and time again, we stumble upon issues not entirely dependent on us, such as missing source data provided by a third-party supplier. This can be either due to **server downtime**, **rebuilding tables**, **schema/object name** **changes**, **missing access rights** or **expired access tokens** given to the service accounts or the analytics team.

![](/_posts/img/1__RcVFxjjOfnewpyaUvdk0JA.png)

When this happens, it’s a [**doomsday machine**](https://en.wikipedia.org/wiki/Doomsday_device) because it’s either [**garbage in, garbage out**](https://towardsdatascience.com/data-quality-garbage-in-garbage-out-df727030c5eb), or entire deployments are halted due to missing object references or test failures. Before we get into the specifics of production gatekeepers and source data replication, let’s go over how we got here.

*   Benn recently [wrote about](https://benn.substack.com/p/data-contracts) how data contracts can prevent us from _“serving bad food from the kitchen”_ with a simple technical (within stack) solutions designed for data teams.
*   David, advocate of data quality, [described](https://davidsj.substack.com/p/yet-another-post-on-data-contracts?utm_source=profile&utm_medium=reader2) an optimal data contract and highlighted important points about the means of communication between parties.
*   Tristan wrote briefly about [his concerns](https://roundup.getdbt.com/p/contracts-have-consequences?r=125hnz&utm_medium=ios&utm_campaign=post) on the responsibility side when the contracts are made.

By no means, I’m an expert in this topic, but if I had to define the term, I would say that it’s the _means needed to bridge the gap between data providers and the data team_. Schemas?! Contracts can also be associated with the emerging trend of [**semantic layer**](https://blog.rittmananalytics.com/the-dbt-semantic-layer-data-orchestration-and-the-modern-enterprise-data-stack-78d9d9ed5c18), which would allow us to **robustly scale our codebase and centralize business metric definitions**. Put simply, elements needed to avoid **communication failure** and to be on the same page.

![](/_posts/img/1__Sl3x8dzxJQIhjFabeuI8gQ.png)

In terms of responsibility, **who’s to blame?** The committee, or the one who launches the doomsday machine? (And even if we find who to blame, the deployment is blown — and users are mad at us…)

My take on data contracts (agreeing with David) is that it was considered normal for a data engineer to fix a broken pipeline. Information can flow through organizations very slowly, especially **when priorities do not align** between the data provider and the analytics team. When that happens, we usually implement a [“**custom fix”**](https://twitter.com/changelog/status/1586882941963091969) **to amend the issue temporarily** and move on with our life.

![](/_posts/img/1__JL6__b7eCFMvyi4cW__rbnYQ.png)

This is completely fine **if we are a fast-paced team**, and we know that fixing the issue takes time. On the flip side, it **hurts the core logic of our data model** by introducing custom (oftentimes not well documented) logic — to be refactored in the future, slowing down future processes.

For myself, I like tackling the tech debt, but it’s hard to convince management that it is worth the investment…

![](/_posts/img/1__Ugt5qqywlpEpVJb2DJDY1w.png)

So what can we do? In the last year, we have seen/developed many creative ways to overcome external dependencies and protect our production data on Snowflake and BigQuery from being infected with bad data.

### ❄️ Snowflake

#### 🟦🟩 Blue/Green Swapping

This is probably the _de-facto_ way for Snowlake and dbt users to deploy to production. Tldr: Blue just becomes green behind the scenes, but nothing about your queries would have to change. I don’t feel like elaborating much on this topic, as [the community](https://discourse.getdbt.com/t/performing-a-blue-green-deploy-of-your-dbt-project-on-snowflake/1349) and [folks from Montreal Analytics](https://blog.montrealanalytics.com/blue-green-deployment-with-dbt-and-snowflake-922f1c658011) have already discussed this profoundly:

⚠️ **Limitations:**

As former `production`tables get demoted to `staging`via the [swap](https://stephenallwright.com/snowflake-swap-table/) feature, the next incremental load will build on top of obsolete tables in staging before swapping.

_Let me break it down to you:_

1.  **T:** Imagine that our run fully refreshes the data in staging, then swaps it with the production database in **T-1**
2.  **T+1:** A new production cycle is due, but only certain data sources have to be processed again, so you decide to go with an incremental refresh.
3.  **T+1:** You then wonder why all the information we updated in **T** is missing?

![](/_posts/img/1__O60__3I04xf__VAf0qUL2yMw.png)

Well, since we built on top of a production data in **T-1** before swapping again, the information loss is the difference between staging and production in **T**!

#### 🧻 Production Rollbacks

Sean has described [another alternative](https://discourse.getdbt.com/t/production-rollbacks/2513?u=sung) for detecting bad data and restoring the previous version, which then later was [demo-ed by Sun](https://www.loom.com/share/bcfd2cf3b4b5471683bfc5b24587db3d)g. Instead of swapping databases, (1) we copy the previous production data to a staging environment, (2) rebuild the tables under scope, then (3) clone it back to production.

This **also works with incremental models,** because we always clone back the latest production loading before building on top of it.

![](/_posts/img/1__3rmAhe3tHzClRz30T5eq5Q.png)

### 🔍 BigQuery

Unfortunately, **swapping is not possible** (or cumbersome) on BigQuery, so the idea is to run & test all models before loading it to the production environment instead. BigQuery is not able to rename datasets, therefore, **swapping by renaming with subsequent commits is ruled out**.

![](/_posts/img/1__dI0Ir__VxWcJWBxkfL4PU4g.png)

#### WAP

[Claus](https://www.linkedin.com/in/clausherther/) has proposed a way a while ago to overcome these limitations. His solution is called [WAP](https://calogica.com/assets/wap_dbt_bigquery.pdf) (Write-Audit-Publish).

1.  Builds all in an audit staging dataset
2.  **Re-build only top level** models under the activation (BI) layer in prod again (testing is optional here).
3.  Traffic is stopped if the build failed in the audit environment.

![](/_posts/img/1__i5ig4efTWXE5QGvTqjgWLg.png)

#### WAC

Since BigQuery [announced](https://cloud.google.com/bigquery/docs/release-notes#February_15_2022) their Table Cloning feature this year, [we](https://hiflylabs.com/) came up with an extended strategy called WAC (**Write-Audit-Clone**) — coined by Hiflylabs.

Instead of rebuilding with an additional cost, let’s **clone the top-level tables** with the `mart` tag from `staging`to `production` !

Note that the prerequisite of cloning top-level tables is that you have a [**well-layered project**](https://www.getdbt.com/analytics-engineering/modular-data-modeling-technique/) either with tags or clear-cut folder structure.

![](/_posts/img/1__mJW2pi8C1byxanK6ARxk9A.png)

The full-blown macro below is able to implement this logic by following the steps below.

1.  Collecting top level tables ( `mart` ,`utils` ) by **querying the** [**graph context variable**](https://docs.getdbt.com/reference/dbt-jinja-functions/graph) for [tags](https://docs.getdbt.com/reference/resource-configs/tags) (you can also use relative paths).
2.  Since sets are not supported in Jinja, we need either create a custom macro for finding any intersections between our chosen tags and tags compiled from the `manifest.json`or check element by element.

![](/_posts/img/1__5rJL7BIjx4KaiJbZr7vGAA.png)

3\. Finally, we `create or replace` filtered tables (we can’t clone views!) in `mart`, `utils`by cloning `analytics_staging`.

Note that there is a custom logic here. First, the macro was written in a way, so it is able to copy between different dataset (=schema) structures.

This has been tested and implemented on projects with [advanced custom schema](https://docs.getdbt.com/docs/building-a-dbt-project/building-models/using-custom-schemas#an-alternative-pattern-for-generating-schema-names) strategies like `generate_schema_name_for_env`.

_If you are using similar building in staging and production with the same custom schemas, feel free to reduce this code to a couple of lines!_

Include list of tags to clone in `project.yml`

vars:  
  tags\_to\_clone: \['mart', 'utils'\]

Then you set the `target.name` to `staging`and call the macro at the end of your production build.

```
#build it in analytics_stagingdbt seeddbt snapshotdbt build
```

```
#copy marts and utils layer to productiondbt run-operation write_audit_publish
```

Voilà, you have a robust, nearly _zero-additional-cost_ way to deploy safely to production!

⚠️ **Limitations:**

*   **We can’t copy views** (not usually the case for objects powering the BI layer)
*   Additional storage (negligible) cost incurred on the difference between clones and source tables
*   More on cloning limitations [here](https://cloud.google.com/bigquery/docs/table-clones-intro#limitations)

### G2K: 🪞Mirror layer

What if we just add another layer to our project that copies the source tables? That means that we are **acting as pseudo data providers**!

Structurally, the project would be extended with a `raw` layer just before staging which copies the source tables 1:1:

![](/_posts/img/1__F5HSaIbQuoyp2FCklT0__vQ.png)

In very simple terms, each model would just include a `select *` from the source and materialize it as a table.

with source as (

    select \* from {{ source('name', 'table') }}

),

This layer then would be separated from the dev/production main build job and **refreshed manually when needed**. Yes, it doesn’t sound right, and oftentimes are not worthy to implement in cases where the data arrives in micro-batches or through streaming.

A _source data refresh job_ would only build (and test) the source data again:

#assuming that you have project tags  
dbt build -s tag:raw

The _production job_ then would build from the `staging` layer.

#build it from staging  
dbt build -s tag:staging+

⚠️ **Limitations:**

*   Reprocessing one additional layer without any change in data → **Additional processing and storage costs**
*   Manual refreshing of source data → loss of automatically provisioned data
*   Can only be used on projects where the **data is “slowly changing”**

### 💡 Why should you care?

Because, w[e](https://hiflylabs.com/) always find ways to make the dbt experience as smooth as possible.

Using WAC in production is not a mere concept, it was well tested, implemented by our team of analytics engineers and to this day, possibly the closest to blue/green deployment for BigQuery users, as it has **no additional storage and compute costs on top of building your production data** (\*insert Snowflake swapping appreciation\*).

Do share your experience with WAC, and don’t hesitate to hit us up if you have any questions! 🥳