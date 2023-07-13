---
layout: single
title: "Delivery semantics: overview"
date: 2023-07-11 04:30:00 +0300
categories: overview messaging delivery-semantics exactly-once at-least-once at-most-once
---

In this article I want to make an overview of a delivery semantics in messaging systems, describe delivery guarantee and
add my own thoughts about all of this.

# Table of contents
1. [Delivery semantics: overview](#delivery-semantics-overview)
   1. [At most once](#at-most-once)
   2. [At least once](#at-least-once)
   3. [Exactly once](#exactly-once)
2. [Conclusion](#conclusion)

# Delivery semantics: overview <a name="delivery-semantics-overview"></a>
So, what exactly is `delivery semantics` and why this is important? `Delivery semantics` is about guarantees provided by messaging system or delivery protocol.
These guarantees are about message order (delivery and processing), delivery reliability, duplication allowance and so on. In other words `delivery semantic` determines how exactly message will be handled in terms of delivery. 

Why this is important? I will try to answer on this question using couple of examples. 

Let's imagine that we develop a system which scrape metrics from other services.
Our system will scrape metrics from each service with some predefined interval, for example, each 15 seconds. After scrapping out system will save these metrics in some storage for future processing.
Each scraped metric is a `message`.

![metrics scrapper example](/assets/images/2023/delivery-semantics/metric-scrapper-example.png)

Assume that other services are not super critical and if metric extraction attempt wasn't successful world will not collapse. 
In this case we can allow our system to `lost` some messages because we know that another attempt will be made in the next `15 seconds`. 
Such guarantee is called `at most once`.

Now assume that we develop a part of a big e-commerce product which responsible for generation and sending of some kind of confirmations like purchase or delivery.

![confirmation service example](/assets/images/2023/delivery-semantics/confirmation-service-example.png)

Sending confirmations is an important process, and we cannot allow our system to lost messages. Each lost message means that customer may not get a confirmation about purchase or delivery, which can lead worsen user experience.
Because of it we made a decision that our service will retry confirmation sending until we are 100% assured that confirmation is sent.  
This guarantee is called `at least once`. As you can see this guarantee already add more complexity in our service because now we have to concern about retries and also we have to store some state for it (otherwise how we know what should be sent?).

The last example will be a banking system which responsible for transactions. 
  