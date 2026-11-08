+++
title = "Upgrading Debezium across SeaTunnel's CDC connectors"
weight = 4
draft = false
project = "Apache SeaTunnel"
+++

SeaTunnel's MySQL, Oracle, PostgreSQL and SQL Server CDC connectors were all on Debezium 1.9.8. Jumping straight to 3.x would have been too risky, because every connector would have had to change at once.

I started by refactoring the build so that each connector can depend on its own Debezium version.