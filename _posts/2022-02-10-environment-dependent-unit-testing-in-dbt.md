---
title: Environment-dependent Unit Testing in dbt
description: >-
  How to build up your unit-testing framework from scratch with dbt using custom
  test macros
date: '2022-02-10T19:27:15.774Z'
categories: []
keywords: []
slug: /@nszoni/environment-dependent-unit-testing-in-dbt-c081a0a5ff1e
---

# Environment-dependent Unit Testing in dbt

![](/_posts/img/1__Oy6__Ti0E__YfqVXlIn2IreQ.jpeg)

I’ve been using dbt for a while now and I feel that most developer needs are covered with community-developed test suite packages like [dbt\_utils](https://github.com/dbt-labs/dbt-utils#schema-tests), [dbt-expectations,](https://github.com/calogica/dbt-expectations) or custom generic tests defined via the mix of Jinja and SQL.

However, **growing business and data requirements demand an automated way of validating our transformations locally.** Cherry-picking observations and manually doing regular check-ups on the transformations can increase the technical debt exponentially, especially when the complex procedures leading to the desired outcome are not trivial to the naked eye.

Lately it has been our priority to find a way of **unit testing our “black boxes”**, and make comparisons between the expected output and the output our sets of models have produced with respect to a static input. This way, we can ensure that the business logic is accurately transformed into code.

There are multiple ways of doing this in dbt, but I found that **existing solutions neither fully exhibit the end-to-end testing, nor achieve the flexibility to run our tests dependent on the development environment.**

So here’s my quick take on solutions suggested by the community:

*   [**Testing with fixed data set**](https://discourse.getdbt.com/t/testing-with-fixed-data-set/564) — Maybe one of the most helpful discussions about unit testing with the capability of defining environment-dependent unit tests, but **doesn’t fully demo the workflow**.
*   [**How to unit test SQL transforms in dbt**](https://www.startdataengineering.com/post/how-to-test-sql-using-dbt/) — Also an excellent resource with an example of running equality checks **misses the flexibility to reduce the test scope to particular columns.**
*   [**dbt-datamocktool**](https://github.com/mjirv/dbt-datamocktool) — packaged version of unit testing where you can define tests in a more structured way without touching the models. Works well, but **lacks the possibility of defining test-specific environments**.

In short, all of the current resources apply great approaches, but my company needed were the following two core features:

1.  **Separate unit-testing from other environments (avoids running in daily jobs)**
2.  **Ability to restrict the target columns in test arguments**

Without further ado, this is how we have built our unit testing environment!

#### **Step 1. Create a unit-testing environment**

This is the environment we will use when defining our tests specifically for validating the transformation logic. How you configure the profile.yml is entirely up to you, just make sure that you **reference this target when testing**. You may mirror the development profile or opt for materializing unit test models in their own schema.

#### **Step 2. Add mock data to the project**

Next, we can now add our input which passes through all the transformations as well as an expected output mock data validating the end-product to the data folder (assuming that our project uses the [default seed path](https://docs.getdbt.com/reference/seed-configs)).

An efficient way of doing a unit test is to **assemble a mock input that raises all the possible edge scenarios the model should be able to handle**. In other words, we should be able to spot-check every component of the business logic.

What we usually do is **pick out a certain entity (account, organization, user, etc.) and manipulate the input accordingly.** This way the comparison will be faster and more concise, rather than just feeding the large chunk of the data. The latter scenario also prompts for a larger expected output calculation.

To keep everything organized, I created a **unit\_testing** folder under the seed folder. This way we can easily identify which seed files are there for mock testing, and which are in the traditional dbt sense. This is useful, especially if you do a lot of unit testing during the development making the project a morass of spaghetti. So, for the sake of your fellow developers, keep things tidy!

![](/_posts/img/0__fzJBSeQ3i1JoYZ__W.jpg)

#### **Step 3. Seed mock data**

Make sure you seed both the input and expected output CSVs to the warehouse, so it will be accessible for the pipeline logic picking up the mock input on the fly and comparing the output to validate your expectations.

![](/_posts/img/0__UKVo2u3q__S5FFath.jpg)

#### **Step 4. Create a reference to the mock input**

Here we tell dbt that **when we are executing a test in the unit testing environment, then instead of picking up the source we are using throughout the project, take the mock data as the input**, otherwise, just proceed with the production staging data.

#### **Step 5. Define your test criteria**

Next, we have to define the test we are planning to execute on the mock output and the model outcome table. In our instance, we wanted to check whether we got the same for a certain input calculated by a spreadsheet tool and dbt. [dbt\_utils has an equality check test](https://github.com/dbt-labs/dbt-utils#equality-source) out-of-the-box but **doesn’t let you switch between environments**.

Therefore, we had to write a custom test which, in essence, **extends the base equality test logic with the possibility of running it only in a given environment**. It is very likely that someone has already thought of similar checks (in our case dbt Labs), but there are a couple of things you want to shuffle. As a result, it is worthwhile checking out the source code behind the open-source test suite packages which can be an excellent reference to build upon additional features.

We simply extracted the relevant components of the default equality test macro and added a conditional statement checking the context we are currently testing.

**It accomplishes the followings:**

1.  **Retrieves the comparison model reference and the specified environment** in the test YAML block (See Step 6.)
2.  If the **context target name matches the environment** for unit testing then  
    **a.** Checks whether there are specific columns defined by the user to be tested. If not, it takes all columns from the schema and concatenates these columns to a variable used later in the select statement  
    **b.** Then executes the comparison between the model output and the expected CSV using the defined columns in (a.).
3.  **If the context name differs** from the defined one, it returns 0, and the test passes

We can store our custom tests as macros or generic tests as per the [dbt documentation](https://docs.getdbt.com/docs/guides/writing-custom-generic-tests).

#### **Step 6. Add custom test to YAML block**

Finally, we can use the custom test in our YAML blocks as the sample below:

We must first **specify our comparative data as well as the context in which the test will be carried out**. We may further reduce the number of columns to check by providing additional arguments.

Ideally, we would define unit tests in separate schema YAML files but unfortunately, dbt does not allow us to have multiple model definitions for schema tests.

#### **Step 7. Build and Profit!**

Now that we have everything set up to do systematic checks in local development, we can simply **run a dbt build command with rerouting the target to the unit test profile.**

It will apply the transformation on the mock input dataset and expose that to the expected output. **If your checks have, you might want to review the utilized logic, but better safe than sorry when pushing false data into production.**

(You might as well want to [store test failures in an audit schema](https://docs.getdbt.com/reference/resource-configs/store_failures) to make debugging easier instead of copy-pasting the compiled test)

![](/_posts/img/0__5XobI91QVkhPSSQj.jpg)

#### **Final Remarks**

In this walkthrough, I have shown you how to set up an environment for unit testing and gave you an example of how we make sure that the correct business value is delivered to our clients through equality checks. The main takeaways are:

1.  Unit tests can provide a **regression framework to minimize the chance of breaking what does work.**
2.  **You may find additional edge cases** which were missed during the development work
3.  It is a good practice **to validate questionable “black box” components** of the pipeline.
4.  It is important to **find the right balance of unit testing** and focus robustness checks on models which effectively create business value, otherwise the project can blow up.
5.  Having a separate environment for unit testing in dbt **does not create overhead in production.**

I hope that you enjoyed this blogpost and feel free to share your testing best practices with us! Also, let us know how you would improve our unit testing framework.