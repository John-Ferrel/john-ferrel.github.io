---
title: Similar-Item Identification and Demand Forecasting Based on Machine Learning
draft: "false"
tags:
  - forecasting
  - machine-learning
created: 2026-03-29 18:15
modified: 2026-03-29 18:15
---
> Why We Do Not Need to Manually Find “Similar Items”: The Model Can Learn Similarity Automatically

---

# 1. Purpose of This Report

In retail forecasting, especially for new-product forecasting, short-lifecycle products, and scenarios where product assortments change rapidly, business stakeholders often raise a very natural question:

> **If a product has very little historical data, or no history at all, how can we forecast it?**
> **Do we have to first find “similar items” and then refer to their sales?**

This question is reasonable in itself.  
Because in many traditional business workflows, “finding similar items” has always been a common approach:

* find similar styles from last year
* find products in the same category and price band
* find products with similar style and seasonality
* use the historical sales of these “similar products” as reference

What this report aims to explain is:

> **Machine learning methods (such as LightGBM and GBDT) do not bypass the problem of similar-item identification. Instead, they upgrade “manually finding similar items” into “automatically learning similarity relationships from data.”**

In other words:

* the traditional approach **explicitly finds similar items by hand**
* the machine learning approach **implicitly learns which products are more similar and uses those historical samples for prediction**

Therefore, the core conclusion of this report is:

> **Machine learning models inherently include the capability of “similar-item identification,” and in most cases they are more objective, comprehensive, and scalable than manual rules.**

---

# 2. Why Business Scenarios Always Need “Similar Items”

In retail forecasting, many products are not suitable for direct item-level time-series extrapolation. There are three main reasons.

## 2.1 New Products Have No Historical Sales

For newly launched products, there is often insufficient historical data, which makes it impossible to directly forecast future sales based on the product’s own past sales.

For example:

* a new Spring 2026 T-shirt
* a newly launched shoe model
* a new color / size combination

In such cases, the business naturally thinks:

> “Then let’s refer to similar products we sold before.”

This is exactly where the idea of “similar-item identification” comes from.

---

## 2.2 Product Lifecycles Are Short

Many retail products have short lifecycles, especially in fashion, fast-moving consumer goods, and promotional products.  
Even if a product is not brand new, its history may still be very short, for example only a few weeks or a few months.

This means:

* the product itself contains very weak temporal patterns
* random fluctuations can be large
* it is difficult to learn stable patterns from the product’s own history alone

Therefore, we also need to borrow information from “other similar products.”

---

## 2.3 Transferable Experience Naturally Exists Across Products

Business experience has always known that:

* products in the same category often have similar selling rhythms
* products in the same price band often have similar demand levels
* products in the same season are affected by similar holidays and weather conditions
* products under the same promotion type often show similar uplift patterns

That is, products are not isolated from one another.  
The historical performance of other products can often provide useful reference for the current one.

This is the essence of “similar-item forecasting”:

> **Use historically more similar products to help forecast the current product.**

---

# 3. Traditional Manual Similar-Item Matching and Its Limitations

In many business teams, similar-item identification usually relies on approaches such as:

* by category
* by price band
* by season
* by style
* by expert judgment such as “this new item looks like that old item”

This approach is intuitive and easy to understand.  
But it also has clear limitations.

---

## 3.1 The Definition of Similarity Is Too Subjective

Manual judgments of “similarity” can easily vary from person to person.

For example, different business stakeholders may give different answers:

* A believes “category is the most important”
* B believes “price is the most important”
* C believes “color / style is the most important”

As a result:

* the same new product may be matched to different “similar items” by different people
* there is no unified standard
* it is difficult to reuse at scale

---

## 3.2 Rules Are Usually Too Few to Express Multi-Dimensional Similarity

In real business scenarios, “similarity” is rarely determined by a single dimension. It is usually the result of multiple dimensions combined.

For example, a product’s sales performance may be influenced simultaneously by:

* category
* brand
* price band
* season
* launch timing
* promotion status
* store type
* regional differences
* lifecycle stage

If we rely only on a few manual rules to find similar items, many important factors will be ignored.

In other words:

> **Similarity in the real world is multi-dimensional, while manual rules usually cover only a small subset of it.**

---

## 3.3 It Is Difficult to Scale to Large SKU Catalogs

When the number of SKUs becomes large, manually finding similar items quickly becomes inefficient:

* it requires substantial manual maintenance
* every new launch needs fresh judgment
* rules easily become outdated when product structures change
* they cannot be continuously updated in a sustainable way

Therefore, manual similar-item methods are usually suitable for:

* small-scale settings
* high-value, low-volume products
* cases with strong dependence on expert knowledge

But not suitable for:

* large SKU catalogs
* frequent new launches
* rapidly changing assortments

---

## 3.4 It Is Hard to Adapt Dynamically to Market Changes

Market conditions change frequently, for example:

* pricing systems change
* consumer preferences change
* channels change
* promotion strategies change
* regional demand changes

Manual rules are usually static, while the business environment is dynamic.  
This means the similar-item rules themselves must also be continuously adjusted.

---

# 4. The Core Idea of Machine Learning: Not Manually Finding Similar Items, but Learning Similarity Automatically

Machine learning methods (such as LightGBM and GBDT) solve the problem in a different way from manual similar-item matching.

Their basic idea is not:

> “First specify a few similar products and then use them as reference”

but rather:

> “Put together the data of all historical products, and let the model learn by itself: under what conditions do products behave more similarly?”

That is, the model does not look only at one product in isolation. Instead, it learns jointly across **all products, all stores, and all time samples**.

This is what people refer to as:

* cross-product learning
* cross-series learning
* a global model
* information sharing

---

## 4.1 A More Intuitive Way to Understand It

You can think of a machine learning model as:

> **a system that automatically summarizes historical experience**

It does not rely on a human to say:

* “A is like B”
* “C is like D”

Instead, from a large number of historical samples, it automatically infers:

* which kinds of products are closer in terms of sales behavior
* which factors determine that closeness
* under different conditions, which historical samples should be referred to

So the model is not “skipping similar-item identification.”  
It is performing similar-item identification at a broader scale and a finer granularity.

---

# 5. Why LightGBM / GBDT Are Naturally Suitable for This Problem

Business stakeholders do not necessarily need to know all algorithmic details, but they do need to understand:

> **Why these models are well suited for retail forecasting and similarity learning.**

There are four main reasons.

---

## 5.1 They Are Good at Handling Tabular Features

Most information in retail business is essentially tabular features, for example:

* product attributes: category, brand, price, specification, color, season
* time attributes: month, week number, holidays, major promotional events
* store attributes: region, store tier, trade area type
* operational attributes: promotion, discount, display, channel
* lifecycle attributes: whether it is new, weeks since launch, selling duration

These data types are very suitable for GBDT / LightGBM models.

---

## 5.2 They Can Automatically Learn Nonlinear Relationships

Many business relationships are not linear. For example:

* a 10% price reduction does not necessarily lead to a fixed proportional sales increase
* the same promotion may work differently across categories
* summer products may have different sales rhythms in northern and southern regions
* new products and mature products may respond differently to promotions

LightGBM can automatically learn these complex relationships, without requiring people to hard-code rules in advance.

---

## 5.3 They Essentially Learn from Samples That Are Similar Under Certain Conditions

This is the most important point.

A tree model does not directly say “which product is the most similar,”  
but it continuously splits samples into more similar paths based on features.

That is:

* conditionally similar products will fall into nearby leaf nodes
* conditionally similar historical samples will jointly influence the prediction
* the model automatically determines which dimensions matter more

This is equivalent to saying:

> **the model automatically searches for similar samples in feature space**

---

## 5.4 They Can Use Data from All Products for Knowledge Transfer

Manual similar-item methods can usually refer only to a few designated products.  
By contrast, a machine learning model can simultaneously use:

* all historical products
* all historical time points
* all store data
* all feature combinations

This means:

> **The model does not rely on a single similar product, but on a group of historical samples satisfying similar conditions.**

Statistically, this is usually more stable and less sensitive to the influence of a few abnormal products.

---

# 6. What “Machine Learning = Automatic Similar-Item Identification” Really Means

Here we need to clarify a key concept:

> **Machine learning does not explicitly output a ranked list of “most similar products,” but it does implicitly perform similar-sample selection.**

In other words, the traditional approach is:

* first find similar items
* then refer to those similar items

The machine learning approach is:

* automatically learn “what is similar” during training
* automatically invoke those similar historical experiences during prediction

So the two approaches aim at a similar goal, but use different methods.

---

## 6.1 Explicit Similar Items vs. Implicit Similarity Learning

### Traditional Approach (Explicit)

1. Manually define “similarity”
2. Select a few similar products
3. Use their historical sales as reference

### Machine Learning Approach (Implicit)

1. Input a large set of product, time, store, and operational features
2. Let the model automatically learn which feature combinations matter more
3. During prediction, automatically refer to the “more similar” group of historical samples

So machine learning can be understood as:

> **automatic similar-item lookup + automatic weighted integration**

---

## 6.2 Why the “Implicit” Approach Is Often Stronger

Because it does not search for the “most similar” product under one fixed standard.  
Instead, it dynamically judges across many dimensions:

* for this product, is season more important or is price more important?
* for this store, is brand more important or is promotion more important?
* at this stage, is the new-product effect more important or the holiday effect?

This is much more flexible than manual methods.

---

# 7. Understanding How the Model Identifies Similar Items Through Decision Trees

Tree models are one of the most intuitive ways to understand this idea.

You can think of a tree as:

> the model repeatedly splitting historical samples into increasingly similar groups based on key features, while gradually correcting the prediction value along each branch.

Below is an illustrative diagram.

---

## 7.1 Decision Tree Illustration

```mermaid
flowchart TD
    A["Base prediction = 20"] --> B{"Is it a summer product?"}

    B -->|Yes| C["+6"]
    B -->|No| D["-5"]

    C --> E{"Is it a men's T-shirt?"}
    E -->|Yes| F["+4"]
    E -->|No| G["-1"]

    F --> H{"Is the price band 99–149?"}
    H -->|Yes| I["+3"]
    H -->|No| J["-2"]

    I --> K["Final prediction = 20 + 6 + 4 + 3 = 33"]
    J --> L["Final prediction = 20 + 6 + 4 - 2 = 28"]
    G --> M["Final prediction = 20 + 6 - 1 = 25"]
    D --> N["Final prediction = 20 - 5 = 15"]
````

---

## 7.2 What This Diagram Is Trying to Show

This diagram does not represent a full production LightGBM model.  
It is meant to illustrate an important business fact.

### First, the model automatically performs conditional grouping

For example:

- one group for summer products
- one group for non-summer products
- one group for men’s T-shirts
- one group for non-men’s T-shirts
- one group for mid-price-band products
- one group for non-mid-price-band products

This is essentially a form of **automatic clustering / automatic grouping**.

---

### Second, the samples become increasingly similar after each split

As the path continues downward:

- first split by season
- then split by category
- then split by price band

The remaining historical samples become increasingly close in key attributes.  
Therefore, the samples in a leaf node can be understood as:

> **a group of historical samples that are more similar under the current conditions**

This is exactly “implicit similar-item identification.”

---

### Third, each step is not only grouping, but also numerical correction

In a tree model, a node is not just a classification decision. It also represents a correction to the prediction value.

For example:

- summer product: $+6$
- men’s T-shirt: $+4$
- mid-price band: $+3$

So the final prediction can be seen as:

$$  
\hat{y} = \text{base score} + \sum \text{path contributions}  
$$

For example, for one path in the above figure:

$$  
\hat{y} = 20 + 6 + 4 + 3 = 33  
$$

This shows that the model is not a black box making arbitrary guesses. It is:

> **step by step correcting the prediction according to the product’s conditions and attributes**

---

# 8. Reinterpreting Tree Models from the Perspective of Similar Items

In traditional business understanding, a “similar item” is often a specific product, for example:

- this new item is like a T-shirt from last year
- this new shoe model is like an older style from last season

But tree models work a bit differently:

> they do not necessarily identify “one single most similar product,” but rather “a group of more similar historical samples”

This group of samples may share conditions such as:

- all are summer products
- all are menswear
- all are T-shirts
- all are in the same price band
- all have similar promotion types
- all are sold in similar stores

So what the tree model refers to is not “one particular SKU,” but:

> **a historical sample set satisfying similar conditions**

From a statistical perspective, this is often more stable than referring to only one similar product.

---

# 9. How to Understand This “Implicit Similarity” Mathematically

From a more technical point of view, we can view the model as a function:

$$  
\hat{y} = f(x)  
$$

where:

- $x$ represents features such as product, time, store, and promotion
- $\hat{y}$ represents the predicted sales / demand

The key point is not the specific form of the function, but rather:

> if two samples have sufficiently similar features $x_a$ and $x_b$, then their model behavior will also tend to be similar

That is:

$$  
x_a \approx x_b \Rightarrow f(x_a) \approx f(x_b)  
$$

Of course, “similar” here is not a simple Euclidean distance.  
It refers to an effective notion of closeness learned by the model in business feature space.

So we can say:

> the essence of what the model learns is “what kinds of products are more similar in feature space”

---

# 10. Why Feature Engineering Matters

If the model is responsible for **automatically learning similarity relationships**, then feature engineering is responsible for answering:

> **What information can the model use to judge similarity?**

Therefore, the more reasonable the feature engineering is, the stronger the model’s ability to identify similar items.

---

## 10.1 Product Features: Defining “What It Is”

For example:

- category
- subcategory
- brand
- price band
- specification
- fabric / style
- color
- size
- gender
- season label

These features help the model determine:

> which historical products this item is more similar to in terms of product attributes

---

## 10.2 Time Features: Defining “When It Is Sold”

For example:

- month
- week number
- holiday
- major promotion events
- weeks since launch
- lifecycle stage

These features help the model determine:

> which historical periods are more similar to the product’s current selling stage

---

## 10.3 Price and Promotion Features: Defining “Under What Commercial Conditions It Is Sold”

For example:

- original price
- discount rate
- whether it is on promotion
- threshold discount / flash sale / bundle promotion
- display / exposure information

These features help the model determine:

> under the current pricing and promotional environment, which historical samples should be referred to

---

## 10.4 Store and Regional Features: Defining “Where It Is Sold”

For example:

- region
- city tier
- store tier
- trade area type
- store customer profile

These features help the model determine:

> for the same product sold in different store environments, which historical samples are more relevant

---

## 10.5 Lifecycle and Inventory Features: Defining “What State It Is In”

For example:

- whether it is a new product
- how many weeks since launch
- whether it was historically out of stock
- whether some sizes are unavailable
- days on sale

These features help the model distinguish:

- whether the true demand is low
- or whether the product sold poorly due to stock-outs
- whether the cold-start stage differs from the mature stage

---

# 11. Why Similar-Item Identification Is Already Embedded in Feature Engineering

Traditional manual similar-item identification essentially says:

- same category
- similar price
- similar season
- similar style

And these judgments correspond exactly to fields in feature engineering.

That is:

> **once the model uses these features, it already has the foundation to judge “who is more similar”**

In other words, feature engineering is not merely an auxiliary step outside the model.  
It is itself the structured expression of similar-item identification logic.

So the relationship can be summarized as:

$$  
\text{manual similar-item rules} \longrightarrow \text{structured features}  
$$

and then the model learns:

$$  
\text{structured features} \longrightarrow\text{predictive relationships}  
$$

So the model does not discard business experience.  
It turns business experience into learnable features.

---

# 12. A Business Example: How the Model Automatically Finds Similar Samples for a New Product

Suppose we want to forecast a new product with the following attributes:

- a new Summer 2026 product
- men’s T-shirt
- price band 99–149
- first week after launch
- with promotional activity
- placed in core stores in tier-1 cities

A traditional approach might be:

- manually find similar men’s T-shirts from last year
- further filter a few old products with similar prices
- manually estimate a reference sales volume

The model-based approach is:

1. convert this new product into features
2. feed it into the unified model
3. let the model automatically search historical data for samples with more similar conditions
4. use these similar historical samples to form a forecast

The automatically referenced historical samples may come from:

- men’s T-shirts from last summer
- products in the same price band
- products under similar promotion intensity
- products in stores of similar tier
- products in similar lifecycle stages

Note that there does not need to be only one “similar item.”  
Instead, there is a set of similar samples.

Therefore, the more accurate statement is not:

> “The model found one similar product”

but rather:

> “The model automatically integrated a group of more similar historical experiences”

---

# 13. Advantages of Machine Learning Compared with Manual Similar-Item Methods

---

## 13.1 More Objective

The model learns patterns from historical data instead of relying on subjective individual judgment.  
Therefore, the definition of similarity is more stable and consistent.

---

## 13.2 More Comprehensive

The model can consider many factors simultaneously, rather than just a few hand-written rules.  
This allows the notion of “similarity” to better match the real business world.

---

## 13.3 More Scalable

When facing a large number of SKUs, many stores, and frequent new launches, the model can still handle them in a unified way.  
There is no need to manually assign similar items for every new product.

---

## 13.4 More Updatable

When market conditions change, the model can continue to update itself using new data,  
rather than relying on manually maintained static rules.

---

## 13.5 More Stable

The model refers to a group of similar historical samples rather than a few individual products.  
This reduces the impact of noise and outliers, making predictions more robust and reliable.

# 14. Key Takeaway

The core idea of this report can be summarized in one sentence:

> **We are not skipping similar-item identification — we are upgrading it from manual selection to automatic learning through machine learning.**

More specifically:

> **Machine learning models do not bypass the need for similar items. Instead, they automatically identify groups of similar historical samples across products, stores, time, and operational conditions, and transfer that knowledge to the current prediction task.**

---

# 15. Conclusion

In retail forecasting — especially for new products, short-lifecycle items, and large-scale SKU management — similar-item identification is a fundamental problem.

Traditional approaches rely on manual experience and static rules.  
While intuitive, they are often limited in:

* objectivity
* multi-dimensional representation
* scalability
* adaptability to change

Machine learning methods (such as LightGBM and GBDT), although they may not explicitly output a list of “similar products,” are able to:

* automatically identify similar samples in feature space  
* learn similarity relationships across multiple dimensions  
* transfer historical knowledge to new prediction targets  

From a methodological perspective:

> **Machine learning does not replace similar-item identification — it systematically generalizes and automates it.**

This is why, in modern retail forecasting systems, machine learning can naturally take on a dual role:

> **similar-item identification + demand forecasting**