+++
title = "Migrations of Debezium's MySQL parser from Positive Technology to Oracle's ANTLR grammar"
weight = 1
draft = false
project = "Debezium"
+++

The MySQL connector parsed DDL with a grammar from Positive Technologies that is no longer maintained, and the ANTLR team planned to deprecate it to avoid having several flavours of MySQL grammar.

I migrated the connector to the Oracle MySQL grammar, which the ANTLR project maintains, and then fixed the DDL parsing edge cases that surfaced. I also wrote up the migration on the Debezium blog.
