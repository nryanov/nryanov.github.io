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
   3. [State](#state)
   4. [Workers](#workers)
   5. [Sources](#sources)
   6. [Sinks](#sinks)
   7. [Converters](#converters)
   8. [Transforms](#transforms)
2. [Example](#example)
3. [Conclusion](#conclusion)

# Kafka-connect: overview <a name="kafka-connect-overview"></a>
Imagine you have a task where you need to fetch some data from a database and incrementally store it in kafka or read the consumed data from kafka and store it in the database.
You can solve both tasks using plain kafka consumer/producer API or even use kafka streams library, 
but if you don't need comprehensive data transformations (e.g. enrichment, stream joining) then you can use Kafka connect for it.

![Kafka connect concepts](/assets/images/2023/kafka-connect-concepts.png)

Kafka connect -- is a framework that allows to stream data from and into the Kafka.

Also, there are concepts like `source` and `sink`.  
`Source` -- is a software system which can produce data. It may be an application, which expose data via API, or database, which produce CDC events or anything else.  
`Sink` -- is a software system which can store or handle data. Like a source, It also can be a database, file system or anything else. It may be even just an API which will be called on each consumed message.  
But initially Kafka connect doesn't know how to extract data from `source` and save it into `sink`. To tell it a special `connector` should be defined.

`Connector` -- is a main logical component of Kafka connect. `Connector` is divided into single or multiple `tasks`.
`Connector` and `task` -- both are logical components. To be able to execute `task` we need something `physical`. In Kafka connect `worker` is responsible for the `task` execution.

## Connectors <a name="connectors"></a>
In Kafka connect `connector` may mean two things:
- `Connector` plugin -- actual jar(s) file(s) of the connector
- `Connector` -- registered connector in Kafka connect

To be able to create a connector in Kafka connect connector's plugin should be installed on all Kafka connect nodes.   
Connector is a logical component which describes data stream in or out of the kafka. Also, connector is responsible for task management.

Kafka connect already has a bunch of pre-installed connectors but if it is not enough you can always use other available open-source connectors or even create one by yourself. 

## Tasks <a name="tasks"></a>
`Task` -- another logical component coordinated by `connector` instance. Task is the main connector's unit of parallelism.
The closest analogy for `task` is a `consumer` in the `consumer group`. They are similar because multiple `tasks` of the single `connector` balance data stream between each other and also may re-balance it
if some `tasks` are failed (but failure should not be a fatal).

If `connector` manage `tasks` and describes `what` should be done, `task` describe actually `how` it should be done. Also, `task` 
is responsible for state storage.

## State <a name="state"></a>

## Workers <a name="workers"></a>
![Kafka connect worker](/assets/images/2023/kafka-connect-worker.png)

## Sources <a name="sources"></a>

## Sinks <a name="sinks"></a>

## Converters <a name="converters"></a>

## Transforms <a name="transforms"></a>

# Example <a name="example"></a>

# Conclusion <a name="conclusion"></a>

