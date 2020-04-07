---
layout: article
title: The High Cost of Splitting Related Data
description: Thoughts on an anti-pattern
---

# The High Cost of Splitting Related Data

Consider the following simple architecture:

<img style="display: block; max-width: 100%; margin: 2em auto;" src="{{ '/img/articles/split/split-original.png' | absolute_url }}" />

The two tables in the database are related. I use 'related' loosely:
there could be a foreign key from one table to another, maybe a shared
identifier. To generalise, it is data that tends to be combined when
queried.

A common anti-pattern I see is to split the data like this:

<img style="display: block; max-width: 60%; margin: 2em auto;" src="{{ '/img/articles/split/split-plus1.png' | absolute_url }}" />

Notice how the relationship between the tables has been pushed up from
the database layer to the application layer.

This is often detrimental to reliability, performance, correctness,
simplicity, flexibility and speed of development.

## The Unreliable Network

Consider this pattern repeated further:

<img style="display: block; max-width: 100%; margin: 2em auto;" src="{{ '/img/articles/split/split-network.png' | absolute_url }}" />

Here we see 11 network requests, 5 databases, and 6 servers, compared
to the two network requests, single database, and server of the original.

If we consider each request to have a 99% chance of success, then the
original will have a 98% (0.99<sup>2</sup>) success rate and this new example will have a
90% success rate (0.99<sup>11</sup>). This gets worse every time the pattern is
extended.

See my article
[Microservices and Availability](/articles/microservices-and-availability/)
for a more detailed argument.

## Loss of Functionality

This approach loses the functionality of the database, such as joins,
filtering, ordering and aggregation. These must be re-implemented
(often poorly) at the application layer.

For example, if two tables requires a simple join your API must fetch
the results of the first table via the first API, find the relevant
IDs, and request them from the second API. If you want to avoid an N+1
query the second API must now support some form of 'multi-fetch'.

It could alternatively be implemented by denormalising the data, but
that comes with its own costs and complexities.

## The Interface Explosion Problem

Changes to the structure of the data can result in multiple changes to dependent APIs.

<img style="display: block; max-width: 100%; margin: 2em auto;" src="{{ '/img/articles/split/split-interfaces.png' | absolute_url }}" />

This can really slow down development and cause bugs!

## Incorrectness

Splitting data into multiple databases loses ACID transactions.

Short of introducing distributed transactions, any consistency between
the tables has been lost and they cannot be updated atomically.

See my article
[Consistency is Consistently Undervalued](/articles/consistency-consistently-undervalued/)
for more thoughts on this.

## Performance Crash

The 'API' is often a HTTP server with a JSON interface. At every step
through the API stack, the TCP, HTTP and JSON serialisation
performance costs must be paid.

Aggregations, filtering and joins performed at the application layer
can also result in over-fetching from the database.

## Why Do Developers Do This?

I think this is often an attempt to contain complexity by
misapplying concepts from Object Orientated Programming.

OOP teaches that data should be private and that there should be a
public interface that operates on that data. Here, tables are seen as
internal data and APIs are seen as a public interface to that
data. Exposing internal detail is a sin!

Without going in to a general critique of OOP, of which there are
already plenty, the problem is that relational data naturally resists
that kind of encapsulation. **Tables are not objects!**

## Valid Use Cases

99% of the time, a prerequisite of having a valid use case for this is
your company having the name 'Google', 'Amazon', or 'Netflix'.

Performance is a valid but rare reason to do this. It is possible that
one part of your data has a wildly different access pattern to the
rest. In that case being able to independently scale or change your
choice of database may be useful enough to overcome the resulting
pain.

In my opinion, this is not a useful method for containing complexity.
I have written [Your Database as an API](/articles/your-database-as-an-api/)
for some thoughts on reducing the complexity of large databases.

My advice is to keep your data together until something is about to
break and there is nothing else you can do. I won't say it's never
appropriate to do it, but splitting in this way has a high cost and
should be a last resort.