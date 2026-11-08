+++
title = "Hardening the Oracle MySQL grammar for production CDC"
weight = 3
draft = false
project = "ANTLR grammars-v4"
+++

This is the ANTLR side of the Debezium MySQL migration. The connector now depends on the Oracle MySQL grammar in ANTLR's shared grammars-v4 repository, so every gap in that grammar becomes a failure to parse a user's schema.

While migrating the connector I found cases where the grammar rejected valid MySQL syntax, and I fixed them upstream instead of patching them in Debezium, so every project that uses the grammar benefits.
