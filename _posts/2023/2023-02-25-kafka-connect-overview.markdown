---
layout: single
title: "Kafka-connect: overview"
date: 2023-02-25 22:50:00 +0300
categories: kafka kafka-connect overview
---

# Table of contents
1. [Kafka-connect: overview](#kafka-connect-overview)
   1. [Connectors](#connectors)
   2. [Tasks](#tasks)
   3. [Workers](#workers)
   4. [Sources](#sources)
   4. [Sinks](#sinks)
   5. [Converters](#converters)
   6. [Transforms](#transforms)
2. [Example](#example)
3. [Conclusion](#conclusion)

# Kafka-connect: overview <a name="kafka-connect-overview"></a>
![Kafka connect concepts](/assets/images/2023/kafka-connect-concepts.png)

Kafka connect -- is a framework that allows to stream data from and into the Kafka.

Also, there are concepts like `source` and `sink`.  
`Source` -- is a software system which can produce data. It may be an application, which expose data via API, or database, which produce CDC events or anything else.  
`Sink` -- is a software system which can store or handle data. Like a source, It also can be a database, file system or anything else. It may be even just an API which will be called on each consumed message.  
But initially Kafka connect doesn't know how to extract data from `source` and save it into `sink`. To tell it a special `connector` should be defined.

`Connector` -- is a main logical component of Kafka connect. `Connector` is divided into single or multiple `tasks`.
`Connector` and `task` -- both are logical components. To be able to execute `task` we need something `physical`. In Kafka connect `worker` is responsible for the `task` execution.

## Connectors <a name="connectors"></a>

## Tasks <a name="tasks"></a>

## Workers <a name="workers"></a>
![Kafka connect worker](/assets/images/2023/kafka-connect-worker.png)

## Sources <a name="sources"></a>

## Sinks <a name="sinks"></a>

## Converters <a name="converters"></a>

## Transforms <a name="transforms"></a>

# Example <a name="example"></a>

# Conclusion <a name="conclusion"></a>

