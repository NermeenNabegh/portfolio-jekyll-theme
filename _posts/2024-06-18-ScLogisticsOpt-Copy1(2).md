---
layout: post
title: "Supply Chain Logistics Optimization"
---
# Supply Chain Logisitics Problem
This problem is shared by Brunel University London in this [Link](https://brunel.figshare.com/articles/dataset/Supply_Chain_Logistics_Problem_Dataset/7558679)

## What Question we are trying to solve in this problem?
How do we plan our daily orders fulfilment?

### Further Explanation: 
The question "how" suggests that this is a prescriptive analysis level problem which would be solved through an optimization model. To add some context this is about a supply chain operational level planning decision(s) which almost always requires some kind of optimization.
Take the simplistic overview below (Figure 1). This diagram illustrates the virtual connection between the four "balloons" or aspects of a supply chain decision and very simply put it explains that to "deflate" one of the balloons, you need to "inflate" another. For example, if we need to minimize (deflate) the missed sales, we need to either increase our inventory, imporve our predicitability and forecasting, OR invest in our capacity for better responsiveness to any new demand requirement.

![Local Image](https://raw.githubusercontent.com/NermeenNabegh/portfolio-jekyll-theme/gh-pages/_posts/The4BalloonsofSupplyChain.png)
<div align="center">(Figure 1)</div>

Now let's review our physical flow (Figure 2) based on the problem scope. For the scope of this problem any order needs to be assigned to:
1. A plant (to make and ship the product in the order)
2. An origin sea port (as it seems the customers are all overseas customers
3. A specific sea freight carrier (contracted service)
4. A destination port (At which it seems like the ownership of the product is transitioned to the customer who assumes responsbility for it then)

Since the data format is available in .xlsx format. I have split it first into 7 csv files as required for this capstone project
But you can find find a brief overview of the data in (Figure 3) here.

![Local Image](https://raw.githubusercontent.com/NermeenNabegh/portfolio-jekyll-theme/gh-pages/_posts/ThePhysicalFlow.png)
<div align="center">(Figure 2)</div>

![Local Image](https://raw.githubusercontent.com/NermeenNabegh/portfolio-jekyll-theme/gh-pages/_posts/Data.png)
<div align="center">(Figure 3)</div>

## Objective function:
Since there is no specific objective function required for this problem but we are only provided with the above shared dataset, we need to make an assumption about what the business is requestimg based on the data provided. 
This could be one of the below likely objectives for such a problem:
1. Mazimize revenue
2. Minimize cost
3. Maxmize the number of fulfilled orders

The list can go on but given the data implying the importance of cost to be gathered and not providing any intel about the price (revenue), we will assume that the objective is minimizing cost

## Decision variables:
Again there is no one and only way to look at this problem or solve it, however from the data it appears that we need to decide for every order on a:
1. Plant/ WH
2. Origin Port
3. Sea freight carrier
4. Destination port

With the above, I decided to solve this problem as a Binary Integer Linear Problem where there are 9215x19x11x9 decision variables denoting a switch on or off for every order/plant/port/freight combination (respectively)

