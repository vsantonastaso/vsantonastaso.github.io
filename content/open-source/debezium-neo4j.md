+++
title = "Bringing graph databases into Debezium with Neo4j"
weight = 2
draft = false
project = "Debezium"
+++

Debezium had no support for graph databases, even though Neo4j is widely adopted and has shipped a Change Data Capture API since version 5.13.

I am building the integration in both directions, and each step has a design document, an implementation and a runnable example. A single message transform maps Debezium change events from a relational database into Neo4j's create, update and delete format. A second converter turns Neo4j change events into the Debezium envelope, so a Neo4j source can feed any Debezium sink such as the JDBC sink. The next step is a Neo4j source connector.
