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
2. [E2E delivery and processing](#e2e-delivery-and-processing)
3. [Conclusion](#conclusion)

# Delivery semantics: overview <a name="delivery-semantics-overview"></a>
So, what exactly is `delivery semantics` and why this is important? `Delivery semantics` is about guarantees provided by messaging system or delivery protocol.
These guarantees are about message order (delivery and processing), delivery reliability, duplication allowance and so on. In other words `delivery semantic` determines how exactly message will be handled in terms of delivery. 

Why this is important? Before answering this question, I will start with a few examples. 

Imagine that we develop a system which scrape metrics from other services.
Our system will scrape metrics from each service with some predefined interval, for example, each 15 seconds. After scrapping out system will save these metrics in some storage for future processing.
Each scraped metric is a `message`.

![metrics scrapper example](/assets/images/2023/delivery-semantics/at-most-once-example.png)

Assume that other services are not super critical and if metric extraction attempt wasn't successful world will not collapse. 
In this case we can allow our system to `lost` some messages because we know that another attempt will be made in the next `15 seconds`. 
Such guarantee is called `at most once`.

Now assume that we develop a part of a big e-commerce product which responsible for generation and sending of some kind of confirmations like purchase or delivery.

![confirmation service example](/assets/images/2023/delivery-semantics/at-least-once-example.png)

Sending confirmations is an important process, and we cannot allow our system to lost messages. Each lost message means that customer may not get a confirmation about purchase or delivery, which can lead worsen user experience.
Because of it we made a decision that our service will retry confirmation sending until we are 100% assured that confirmation is sent.  
This guarantee is called `at least once`. As you can see this guarantee already add more complexity in our service because now we have to concern about retries and also we have to store some state for it (otherwise how we know what should be sent?).

The last example will be a banking system which responsible for transactions.

![confirmation service example](/assets/images/2023/delivery-semantics/exactly-once-example.png)

In this example there is some `money transfer service` which responsible for a full transaction cycle:
- Get money from first user's account
- Add money to the second user's account

Each step must be done exactly once, otherwise we will have inconsistent state:
- If money will be withdrawn twice from the first user's account, then this user will lose some money
- If money will be added twice to the second user's account, then, probably, second user will be happier than usual as he will get more money, but, at least, it will definitely affect net income

Also, both problems, at worst scenario, may cause problems with regulatory.
To guarantee that everything goes well our `money transfer service` must do both operations strictly one time. Such guarantee is called `exactly once`.

These examples showed that globally there are three guarantee types:
- `at-least-once` -- some messages may be lost
- `at-most-once` -- some messages may be duplicated
- `exactly-once` -- no messages lost and no messages duplicated

So, why delivery semantic is important? Why we can't just always select the highest guarantee and develop every system where everything processed `exactly-once`?  
As you can see from examples above, different scenarios require different solutions. Also, the higher guarantee you choose, the higher the cost of development in terms of complexity.

In the next paragraphs I will explain each guarantee more profoundly. After that we will discuss end-2-end delivery and processing.

## At-most-once <a name="at-most-once"></a>
`At-most-once` is the weakest message delivery guarantee. As we already know services which implement it may lose messages, but they shouldn't produce duplicates.
In real live services may as lose messages as produce duplicates of them, but let's imagine that only message lose is possible for now.

So, globally we have two situations:
- Message was delivered successfully
- Message was lost during delivery

There are other scenarios where message may be lost, but, again, for now imagine that our `consumer` always handle messages correctly if at least get them.

First case is `happy-path` and shown in the next picture:

![confirmation service example](/assets/images/2023/delivery-semantics/at-most-once-part-1.png)

Here `producer` produced message and successfully sent it to the `consumer`. `Consumer` also successfully handled it.

Unfortunately, in production systems we also have to consider not only `happy-path`. What can go wrong?
Actually, everything, starting from `producer`:
- `producer` may don't even send message because of different reasons (e.g. bugs)
- `producer` may send message, but network isn't 100% reliable and message may be lost between `producer` and `consumer`

`Consumer` may also not be able correctly handle message due to errors, but as I defined previously, let's think that for now `consumer` is reliable and always correctly handle messages.

![confirmation service example](/assets/images/2023/delivery-semantics/at-most-once-part-2.png)

If `at-most-once` is such unreliable that can lose messages why it even exists? 
There are multiple answers for it:
- Because of nature of some systems
- Implementation simplicity

What does `nature of some systems` mean? It means that some systems have special requirements like delivery speed rather than reliability.
A classic example is VoIP (VoIP) services. Such services use UDP to transfer data. `UDP` is a transport protocol that does not guarantee
delivery of packages (order and absence of duplicates are also not guaranteed), but using of this protocol allows continuing to receive new data (or voice in this example). 
At the end, repeating last sentence(s) to your companion is not a big deal in comparison with a stopped call.

What about `simplicity`? `Simplicity` in case of `at-most-once` means that it may not implement retries, may not support ordering and may just `send and forget`. 
Such systems are much easier to implement, because of almost no delivery guarantees.

Believe me, this is a miracle if you can choose this guarantee, but it's an uncommon case, at least, in my practice.

Now let's move on to the next guarantee: `at-least-once`.

## At-least-once <a name="at-least-once"></a>

## Exactly-once <a name="exactly-once"></a>

# E2E delivery and processing <a name="e2e-delivery-and-processing"></a>

# Conclusion <a name="conclusion"></a>