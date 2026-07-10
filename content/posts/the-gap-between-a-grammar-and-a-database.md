+++
title = "The Gap Between a Grammar and a Database"
date = "2026-07-09T00:00:00Z"
draft = false
summary = "What migrating a MySQL DDL parser between ANTLR grammars taught me about the structural gap between grammar specifications and real database behavior."
categories = ["posts"]
author = "Vincenzo Santonastaso"
+++

Parsing SQL is a solved problem until you try it.  
There are grammars, there are parser generators, and there are decades of literature on formal languages.  
You feed a grammar to ANTLR or Yacc, generate the related parser and that's all.  
This is what I thought when I had to migrate a MySQL DDL parser from the Positive Technologies ANTLR grammar to Oracle's official one.  
In a CDC platform, for example, we use a parser to track how table definitions evolve over time by processing every *CREATE TABLE, ALTER TABLE, DROP TABLE*, or any other DDL statement exactly the way MySQL should.  
Get it wrong, and the parser builds an incorrect schema, which means wrong data downstream, and nobody notices until it's too late.  
The Positive Technologies grammar had been deprecated ([https://github.com/antlr/grammars-v4/pull/4443](https://github.com/antlr/grammars-v4/pull/4443)) in the *grammars-v4* repository in favor of Oracle's official MySQL grammar, so the migration was inevitable.  
I figured I'd swap the *.g4* files, fix a few imports, and be done in a couple of days but that's not what happened.   
Over the course of the migration, in addition to all the fixes needed on the platform, I found several potential bugs in Oracle's official grammar, cases where valid SQL that runs fine on official MySQL engines was rejected by the grammar.  
I fixed all of them upstream across multiple pull requests, but we will talk later about the process of finding and fixing them.  
What surprised me wasn't the bugs themselves. It was what they revealed about the distance between a grammar specification and how a database actually behaves.   
That gap is wider than I expected, and it's structural, not accidental.  
I try to dive deep into that gap: where it comes from, why it's hard to close, and what working inside it taught me about parsing real-world SQL.

## Grammars and parsers in practice

At a high level, a grammar is just a description of what valid syntax is supposed to look like and SQL turns out to be a pretty good example of why this matters.   
The language is deeply nested, overloaded with keywords, and full of cases where the same token means different things depending on context.   
So instead of matching text directly, you usually generate a parser from a grammar definition.  
In practice there are normally two stages involved:  
first comes lexing: turning raw text into tokens.

Something like:

```sql
CREATE TABLE foo (id INT)
```

gets split into a token stream roughly like:

*[CREATE] [TABLE] [foo] [(] [id] [INT] [)]*

After that, the parser consumes those tokens and tries to recover structure from them.   
Not just "these are words," but "this is a CREATE TABLE statement," "this identifier is the table name," and so on.  
Once you start building real tooling around SQL, the parser generator you choose ends up affecting much more than syntax.   
It changes how grammars are written, how ambiguity gets resolved, what debugging looks like, even which programming languages you can realistically target.

## Yacc/Bison vs ANTLR

If you spend enough time around database tooling, you eventually run into two ecosystems over and over again: Yacc/Bison and ANTLR.  
They solve roughly the same problem, but they come from very different eras and make very different tradeoffs.  
Yacc originated at Bell Labs in the 1970s. GNU Bison is the version most projects actually use today.   
MySQL's parser is built on Bison (*sql_yacc.yy*), and PostgreSQL uses it too through *gram.y*.  
Some of these grammar files are older than the engineers maintaining them.  
ANTLR ([https://www.antlr.org/](https://www.antlr.org/)) arrived much later when Terence Parr released the original version in the early 1990s, and ANTLR4 showed up in 2013 with a fairly different parsing strategy from traditional Yacc-style tools.  
The distinction matters almost immediately once the grammar becomes nontrivial.

### Parsing strategy

Yacc/Bison and ANTLR differ most in how they make parsing decisions.  
Yacc and Bison are bottom-up parsers (LALR) so they consume tokens as they come in and only later decide how those tokens fit together.  
ANTLR works the other way around. It starts from the grammar and tries to predict which rule is going to match before too much input has been consumed.  
That difference can feel a bit abstract until you run into a real SQL grammar.  
Take a statement like:

```sql
ALTER TABLE users DEFAULT CHARSET utf8mb4;
```

Here, *DEFAULT* is part of a table option.  
Now look at this one:

```sql
ALTER TABLE users ALTER COLUMN name SET DEFAULT 'unknown';
```

The same keyword appears again, but it means something completely different.  
For a human reader the distinction is obvious after a quick glance but the parser has to work a little harder.  
In a Yacc grammar, the interesting moment comes when the parser reaches *DEFAULT*.   
At that point it may have enough information to reduce what it has already seen, or it may need to keep reading before it can decide which rule applies.   
That's where shift/reduce conflicts come from.  
They're not necessarily a disaster, but in large SQL grammars they tend to show up regularly because so many keywords are reused in different contexts and a keyword that acts as a table option in one statement might be part of an expression or a column definition somewhere else.  
ANTLR approaches the same situation differently with its Adaptive LL algorithm ([https://www.antlr.org/papers/allstar-techreport.pdf](https://www.antlr.org/papers/allstar-techreport.pdf)).   
Rather than forcing an immediate choice, it can keep looking ahead and If the next tokens indicate a table option, it follows that path.   
If they indicate a column default, it follows another.   
In this way the parser keeps gathering information until one interpretation clearly makes sense.  
Most of the time this means fewer explicit conflicts in the grammar itself and when lookahead isn't enough, ANTLR lets you add semantic predicates, small checks that help the parser choose between otherwise valid alternatives.  
The tradeoff appears in another area: left recursion.  
Expression grammars are where this usually becomes visible.  
Suppose you want to describe arithmetic expressions, a natural rule might say that an expression can contain another expression plus another expression:

*expression -> expression '+' expression*

or:

*expression -> expression '\*' expression*

A bottom-up parser like Yacc handles this style naturally since it reads the tokens first and only later groups them into larger structures, so recursive rules like these fit the parsing model quite well.  
A top-down parser has a harder time.  
If ANTLR starts trying to match an expression and the first thing it sees in the rule is another expression, it immediately calls the same rule again.  
That new invocation sees the same thing and calls itself again.  
Nothing has consumed any input yet, so the parser ends up recursing forever.  
ANTLR4 solves a large part of this problem automatically.   
Indeed common cases such as arithmetic and comparison expressions generally work without any special effort because the parser rewrites simple forms of left recursion internally.  
The cases that still hurt are usually the messy ones: precedence rules mixed with optional clauses, contextual keywords, nested subqueries, and dialect-specific SQL features.   
At that point you often find yourself restructuring parts of the grammar manually, regardless of how elegant the original rule looked on paper.

### Grammar structure

The grammars also look very different in practice.

A simplified Yacc rule might look like this:

```c
create_table:
        CREATE TABLE table_name '(' column_list ')'
        {
            $$ = new_create_table_node($3, $5);
        };
```

The important detail there is the embedded C block.   
Grammar rules and semantic actions live together in the same file.   
Some people hate this because it mixes parsing logic with implementation details but on the other hand, when debugging a parser, having everything in one place is occasionally convenient.  
ANTLR grammars separate those concerns more aggressively:

```antlr
   createTable
        : CREATE_SYMBOL TABLE_SYMBOL tableName
          OPEN_PAR_SYMBOL columnDefinition (COMMA_SYMBOL columnDefinition)*
          CLOSE_PAR_SYMBOL;
```

The grammar stays declarative, while semantic behavior moves into listeners or visitors generated by ANTLR. During tree traversal you get callbacks like enterCreateTable() and exitCreateTable()  
Architecturally this is cleaner. Whether it stays cleaner in large projects is a different question. One issue I kept running into during migrations was listener logic drifting away from the grammar over time until the relationship between the two became surprisingly hard to follow.

If you want to see the difference firsthand, both tools are available on Ubuntu 24.04:

```shell
sudo apt install bison flex antlr4
```

Here is a minimal Bison grammar (*sql.y*) with its Flex lexer (*sql.l*):

```c
 /* sql.l */
    %{ #include "sql.tab.h" %}
    %%
    CREATE      { return CREATE; }
    TABLE       { return TABLE; }
    INT         { return INT_T; }
    [a-z_]+     { return ID; }
    [(,);]      { return yytext[0]; }
    [ \t\n]     ;
    %%

    /* sql.y */
    %token CREATE TABLE INT_T ID
    %%
    stmt : CREATE TABLE ID '(' cols ')' ';' { printf("ok\n"); }
    cols : col | cols ',' col
    col  : ID INT_T
    %%
    int main() { return yyparse(); }
    void yyerror(const char *s) { fprintf(stderr, "%s\n", s); }

```

Build and run:

```shell
    flex sql.l && bison -d sql.y && gcc -o parser sql.tab.c lex.yy.c -lfl
    echo "CREATE TABLE users (id INT, name INT);" | ./parser
    ok
```

The equivalent ANTLR grammar (*Sql.g4*):

```antlr
 grammar Sql;
    stmt : CREATE TABLE ID '(' cols ')' ';' ;
    cols : col (',' col)* ;
    col  : ID INT ;
    CREATE : 'CREATE' ;
    TABLE  : 'TABLE' ;
    INT    : 'INT' ;
    ID     : [a-z_]+ ;
    WS     : [ \t\n]+ -> skip ;

```

Generate and test:

```shell
    antlr4 Sql.g4 && javac Sql*.java
    echo "CREATE TABLE users (id INT, name INT);" | grun Sql stmt -tree
    (stmt CREATE TABLE users ( (cols (col id INT) , (col name INT)) ) ;)
```

Both parse the same input. The Bison version prints a short confirmation. The ANTLR version shows you the entire parse tree, which is where grun becomes useful for debugging grammar issues.

### Error recovery

Error handling is another place where the differences become obvious quickly.  
Traditional Yacc-style parsers tend to recover aggressively by discarding tokens until parsing can resume at some known synchronization point.   
Sometimes that works fine but you could get one useful syntax error followed by a cascade of nonsense because the parser lost its place twenty tokens earlier.  
ANTLR4 generally does a better job recovering locally.   
It can often insert or remove a token and continue parsing without destabilizing the entire parse state which matters a lot when processing large SQL dumps or migration files where stopping at the first malformed statement is not acceptable.

### Tooling

The tooling story around ANTLR is also much better than what older parser generators traditionally offered.  
ANTLR ships with utilities that let you inspect parse trees interactively, and IDE plugins can visualize grammar behavior while editing rules.   
Being able to see token streams and parser decisions directly saves an enormous amount of time once grammars become complicated.  
I remember spending hours debugging why *_UTF8MB4'text'* was being tokenized incorrectly. With ANTLR tooling, the issue was visible almost immediately once the token stream was printed out.  
With older Yacc workflows, debugging often degenerates into enabling *yydebug*, scattering *printfs* through grammar actions, and hoping the parser state output eventually reveals something useful.

## How databases maintain their grammars

The way SQL databases manage their grammars is surprisingly inconsistent.  
If you only ever use SQL from the outside, you could suppose that every database does roughly the same thing internally.  
In reality, the parser architecture varies quite a bit, and those choices end up affecting anyone who needs to build tooling on top of the database.   
Looking at PostgreSQL, SQLite, and MySQL side by side makes those trade-offs easier to see and understand the trade-offs.

### PostgreSQL and SQLite: one grammar, one parser

PostgreSQL takes a fairly direct approach.   
In fact the parser grammar lives in the server source tree as *gram.y* ([https://github.com/postgres/postgres/blob/master/src/backend/parser/gram.y](https://github.com/postgres/postgres/blob/master/src/backend/parser/gram.y)), a large Bison grammar that has grown over decades, and the lexer sits next to it in *scan.l*.   
Together they are the parser, so there isn't a separate specification maintained somewhere else.   
When a PostgreSQL developer introduces a new SQL feature, the parser changes usually happen alongside the execution-layer changes, so the grammar and the implementation evolve together because they're part of the same codebase.  
From a maintenance perspective that's hard to beat.   
There's no second parser implementation that can slowly drift out of sync and the downside only becomes obvious when you want PostgreSQL parsing outside the server itself.  
Most projects end up using *libpg_query* ([https://github.com/pganalyze/libpg_query](https://github.com/pganalyze/libpg_query)), which essentially takes PostgreSQL's own parser and packages it as a standalone library.   
If PostgreSQL accepts a statement, the parser will too since you're running the same code. The trade-off is that you're still dealing with a C library and If your application is written in Java, Python, or another language, you now need bindings and all the extra plumbing that comes with them.  
There are also ANTLR grammars for PostgreSQL maintained by the community.   
Some are quite good, some less so.   
The problem is that none of them are authoritative and once a grammar is maintained separately from the database itself, keeping it perfectly aligned becomes an ongoing task rather than a guarantee.

SQLite ends up in a very similar place, although it gets there differently.   
Instead of Bison, SQLite uses Lemon ([https://github.com/sqlite/sqlite/blob/master/tool/lemon.c](https://github.com/sqlite/sqlite/blob/master/tool/lemon.c)), a parser generator created by Richard Hipp.   
The overall philosophy is the same: the grammar and the database live in the same project and evolve together, so there is no separate parser ecosystem that needs to stay synchronized with the implementation.

### MySQL: two grammars, one language

MySQL is where things get more interesting.   
The server parser is based on a Bison grammar called *sql_yacc.yy* ([https://github.com/mysql/mysql-server/blob/8.0/sql/sql_yacc.yy](https://github.com/mysql/mysql-server/blob/8.0/sql/sql_yacc.yy)), which is part of the MySQL source code and ultimately defines what the server accepts as valid SQL.   
At the same time, Oracle also publishes an official ANTLR grammar ([https://github.com/antlr/grammars-v4/tree/master/sql/mysql/Oracle](https://github.com/antlr/grammars-v4/tree/master/sql/mysql/Oracle)) for external tooling like MySQL Workbench that relies on those *.g4* files for parsing, syntax highlighting, and editor features.  
That arrangement solves a real problem.   
Tool developers don't have to embed part of the MySQL server just to understand SQL. Generating a parser from ANTLR files is considerably easier than integrating a large C codebase.   
But the convenience comes with a cost since there are now two independent descriptions of the same language, maintained by different people, released on different schedules, and often tested in different environments.   
Most of the time they agree but sometimes they don't and those disagreements are where things start getting interesting.  
The ANTLR grammar also has requirements the server grammar doesn't.   
MySQL Workbench needs to understand SQL from multiple MySQL versions, so the grammar contains version-specific predicates that enable or disable rules depending on the target server version.   
It's a clever solution, but it introduces another layer of complexity.   
The grammar is no longer describing only SQL syntax but is also encoding version-awareness and one of those predicates ended up causing a real issue during my migration work, which I'll come back to later.

### Community alternatives

Outside the official grammars, there's a surprisingly large ecosystem of SQL parsers.   
The *antlr/grammars-v4* repository ([https://github.com/antlr/grammars-v4](https://github.com/antlr/grammars-v4)) is probably the first place most people look.   
Over the years it has become the default collection of community-maintained ANTLR grammars, covering dozens of programming languages and several SQL dialects.  
For MySQL for example, the repository has hosted multiple grammars.   
For a long time the grammar contributed by Positive Technologies was the one many projects adopted. It was relatively easy to integrate, reasonably well maintained, and worked well enough for a lot of practical use cases.  
But it also accepted statements that the MySQL server itself would reject, which is exactly the kind of discrepancy parser authors try to avoid.  
Eventually it was deprecated ([https://github.com/antlr/grammars-v4/pull/4443](https://github.com/antlr/grammars-v4/pull/4443)) in favor of Oracle's official grammar.

Tree-sitter ([https://github.com/tree-sitter/tree-sitter](https://github.com/tree-sitter/tree-sitter)) approaches the problem from a completely different angle.  
Its primary goal isn't building rich parse trees for compilers or database tooling but is helping editors stay responsive while users type.   
Instead of reparsing an entire file after every keystroke, Tree-sitter updates only the affected parts of the syntax tree.   
For syntax highlighting, navigation, and editor features it's excellent. For extracting detailed schema information from complex SQL, I still find ANTLR's visitor and listener model easier to work with.

Then there's jOOQ, which goes in yet another direction.   
Rather than generating parsers from grammar files, jOOQ implements its parser entirely by hand ([https://github.com/jOOQ/jOOQ/blob/main/jOOQ/src/main/java/org/jooq/impl/ParserImpl.java](https://github.com/jOOQ/jOOQ/blob/main/jOOQ/src/main/java/org/jooq/impl/ParserImpl.java)).   
Parsing rules are expressed directly as Java code and this gives the maintainers an enormous amount of control, and considering that jOOQ supports dozens of SQL dialects, that flexibility is understandable.   
But the price is maintenance.   
In fact every new SQL feature, dialect extension, or standards update eventually becomes code someone has to write, test, and maintain manually.   
It's almost the opposite philosophy from grammar-driven parser generation: less automation, more control, and significantly more responsibility over time.

### What breaks when you switch grammars

Every external grammar is essentially a translation of the server's Bison grammar and may change with any new commit.   
Since these external grammars are maintained independently by different people and on different schedules, they don't always stay up to date.   
When I migrated from the PT grammar to Oracle's, I went from a 100% test pass rate to 19% overnight.   
We can summarize the problems I encountered in three main categories:

*Leniency vs strictness*: the PT grammar accepted SQL that MySQL itself would reject, like NVARCHAR without a length just to give an easy example. That feels convenient until your parser silently builds wrong data models from invalid DDL.   
The Oracle grammar is stricter, and that strictness surfaced years of bad assumptions in my test data.

*Real-world syntax quirks*: old DDL dumps use double quotes as string delimiters (*ENGINE "InnoDB"* for example), but MySQL's default treats double quotes as identifier delimiters ([https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html#sqlmode_ansi_quotes](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html#sqlmode_ansi_quotes)).   
The official grammar enforces the standard, so production DDL that has been running for years suddenly fails to parse.   
The cleanest approach is a preprocessing layer that normalizes the input before it reaches the grammar, rather than polluting the grammar with special cases.

*Context-sensitive keywords*: words like *LAG*, *LEAD*, and *RANK* ([https://dev.mysql.com/doc/refman/8.0/en/keywords.html](https://dev.mysql.com/doc/refman/8.0/en/keywords.html)) are window function keywords in one context and perfectly valid column names in another.   
MySQL is aggressive about allowing keywords as identifiers, but the grammar had them classified as keywords only.   
You find out by running 

```sql
CREATE TABLE t (rank INT) 
```

on a real database and seeing that it works fine, even though the grammar rejects it.

The most pervasive breakage wasn't in the grammar at all but was in the listener code that walks the parse tree.   
When you switch grammars, rule names change, tree shapes change, and every assumption your code made about the structure breaks silently.   
Your grammar is half the parser and the listener code is the other half, and it's tightly coupled to a tree shape that will change.

These bugs aren't isolated mistakes but are symptoms of a structural problem: grammar drift.   
The server grammar (*sql_yacc.yy*) evolves with every MySQL commit and is maintained by the same team that writes the SQL execution code.   
When MySQL adds *GEOMCOLLECTION* as a synonym for *GEOMETRYCOLLECTION* ([https://dev.mysql.com/doc/refman/8.0/en/gis-class-geometrycollection.html](https://dev.mysql.com/doc/refman/8.0/en/gis-class-geometrycollection.html)), the server grammar gets the change as part of the same patch.   
The ANTLR grammar is a separate artifact, maintained separately, and it only gets updated when someone notices the gap.  
Synonyms accumulate over MySQL versions, and the grammar needs to track each one, but there's no automated way to discover them.  
Case sensitivity assumptions, for example, in lexer patterns go unchecked because the test corpus doesn't include uppercase charset introducers.  
Nobody writes tests for *_UTF8MB4* when *_utf8mb4* already passes.   
Version predicates are point-in-time assertions that nobody revisits when the underlying behavior stays stable across versions.  
To make this concrete, here is what actually happens when you feed valid MySQL to the grammar before the fixes.

A synonym the grammar doesn't know about:

```
Input:  CREATE TABLE t (g GEOMCOLLECTION);
ANTLR:  line 1:18 no viable alternative at input 'GEOMCOLLECTION'
MySQL:  Query OK, 0 rows affected (0.02 sec)
```

A charset introducer in uppercase:

```
Input:  SELECT _UTF8MB4'test';
ANTLR:  line 1:7 token recognition error at: '_UTF8MB4'
MySQL:
+----------------+
| _UTF8MB4'test' |
+----------------+
| test           |
+----------------+
```

A version predicate that blocks valid syntax:

```
Input:  SET CHARSET DEFAULT;
ANTLR:  line 1:12 no viable alternative at input 'DEFAULT'  (version >= 8.0.11)
MySQL:  Query OK, 0 rows affected (0.00 sec)
```

In each case, MySQL accepts the statement without complaint.   
The grammar rejects it and the gap between the two is the problem.

This kind of drift isn't unique to MySQL.   
Any language with multiple parser implementations will experience it.   
The server grammar is the ground truth because it's compiled into the binary that actually executes SQL.   
Everything else is an approximation, and approximations decay over time unless someone actively maintains them and the interesting question isn't whether an external grammar has bugs but how you find them.   
In my case, the answer was a large test suite of real-world DDL statements.   
Without that, these bugs could have gone unnoticed for years.

## What I took away

### Test against real databases, not documentation. 

Documentation describes intent but the database describes behavior.   
When I spun up MySQL instances with different versions in Docker and ran the SQL that the grammar rejected, every case succeeded.   
The documentation didn't help me find these bugs but only running the actual SQL did.  
 I know that this seems obvious, but it's easy to skip when you're focused on the grammar itself, *docker run mysql:8.0.46* takes thirty seconds.   
Verifying a parsing assumption against a real database takes less time than debugging a wrong assumption later.

### Contribute fixes upstream. 

I could have patched all six bugs locally and moved on. But the whole point of using an official grammar is that it's correct for everyone, not just for you. If you find a gap between the grammar and the database, fixing it upstream is the right thing to do and also protects you from losing your patches the next time you pull new grammar changes.   
Both of my PRs ([https://github.com/antlr/grammars-v4/pull/4827](https://github.com/antlr/grammars-v4/pull/4827), [https://github.com/antlr/grammars-v4/pull/4834](https://github.com/antlr/grammars-v4/pull/4834)) were reviewed and merged within a few days.   
The grammars-v4 maintainers were responsive and appreciated the detailed verification against real MySQL instances.

### Strictness is a feature. 

A permissive grammar feels convenient since it parses more input, your tests pass, everything looks fine. But a grammar that accepts SQL the database would reject is silently lying to you. The Oracle grammar's strictness exposed problems in my test data that had been hiding for years under the old grammar. Every failure was an opportunity to fix something that would have eventually caused a real bug. I started the migration at a 19% test pass rate and ended at 100%. That delta represents years of accumulated incorrect assumptions that the permissive grammar had quietly papered over.

### Be intentional about which grammar you build on. 

There are multiple ANTLR grammars for MySQL. The Positive Technologies grammar was popular and convenient, but it's now deprecated and unmaintained while Oracle's is actively maintained alongside MySQL releases. Community grammars in the grammars-v4 repository vary widely in quality and currency. The choice of grammar is an infrastructure decision with long-term consequences. Treat it like choosing a dependency. Who maintains it? How often is it updated? What's their process for tracking changes in the database? These questions matter more than whether the grammar parses your test data today.

### Your listener code is part of the parser. 

I spent more time fixing listener bugs than grammar bugs since the code that walks the parse tree and extracts meaning is tightly coupled to the grammar's tree shape, even though it lives in separate files. When the grammar changes, the listener code breaks in subtle ways: wrong column references, missing default values, transposed types. If you're building on ANTLR, treat your listener code with the same rigor as the grammar itself. It's not application code that uses a parser. It is the parser.

### Grammar is a living document. 

Like any specification, it needs the same ongoing care and testing as the software it describes. The gap between a grammar and a database is never zero, but it doesn't have to be wide. You just have to keep measuring it.
