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
   5. [Sources and Sinks](#sources-and-sinks)
   6. [Converters](#converters)
   7. [Transforms](#transforms)
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
State -- is a crucial part for every streaming processing. As mentioned earlier, `task` is responsible for state storage.
The important thing is that `task` doesn't store anything in itself -- all state changes are stored in special kafka topics which are configured 
for kafka connect cluster. 

Distributed state consists of multiple parts:
- Offset storage (`offset.storage.topic`) -- place for storing processed offsets. It is not only offsets of kafka topics but also may be position of last processed row from DB or something like this which will indicate a point from which task can continue 
- Config storage (`config.storage.topic`) -- is a place for store configs of connectors
- Status storage (`status.storage.topic`) -- this is a status of tasks

Standalone state consists of:
- Offset storage -- in case of standalone mode it is a local file (`offset.storage.file.filename`)

Depending on Kafka connect cluster mode (`distributed` or `standalone`) state will be stored differently:
![Kafka connect task state storage](/assets/images/2023/kafka-connect-task-state-storage.png)

If kafka connect cluster running in a distributed mode then state will be stored in the kafka cluster itself,
otherwise it will be stored in the local file system.

## Workers <a name="workers"></a>
Previously we discussed logical components such as `connectors` and `tasks`. But those components should be run somewhere.
`Worker` -- is a physical component which is responsible for running logical components. Simply, it is just a node which is part of kafka connect cluster. 

![Kafka connect worker](/assets/images/2023/kafka-connect-worker.png)

Kafka connect cluster consists of worker nodes, and you can add or remove them almost dynamically. The only thing you should remember is
that each node should have the same:
- `group.id`
- `offset.storage.topic`
- `config.storage.topic`
- `status.storage.topic`

Workers nodes will automatically discover each other if they have the same `group.id` value.

## Sources and Sinks <a name="sources-and-sinks"></a>
`Source` -- is a connector type which responsible for consume data from the outer system and store it into the selected kafka topic(s).  
`Sink` -- is a connector type which responsible for consume data from the selected kafka topic(s) and store these data into the outer system.

As you can see, both `source` and `sink` are connectors. Kafka connect has some pre-installed connectors but as mentioned earlier you can also write your own or install open-source connector.
Installing of the connector is a simple process: you just need a bunch of JAR files required for connector and put them into desired location of each worker node. Then you should update configuration
and specify that location to allow nodes discover and use it.

## Converters <a name="converters"></a>

## Transforms <a name="transforms"></a>

# Example <a name="example"></a>

# Conclusion <a name="conclusion"></a>

