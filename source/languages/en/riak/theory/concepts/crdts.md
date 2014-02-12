---
title: Datatypes
project: riak
version: 2.0.0+
document: appendix
toc: true
audience: intermediate
keywords: [appendix, concepts]
---

A pure key/value store is completely agnostic toward the data stored within it. Any key can be associated with values of any conceivable type, from short strings to large JSON objects to video files. Riak began as a pure key/value store, but over time it has become more and more aware of the data stored in it through features like [[secondary indexes]], [[search capabilities|Riak Search]], and [[counters]].

In version 2.0, Riak continued this evolution by introducing a series of eventually convergent **datatypes** inspired by academic research on convergent replicated datatypes (CRDTs), most notably the work of Shapiro, Preguiça, Baquero, and Zawirski ([paper](http://hal.upmc.fr/docs/00/55/55/88/PDF/techreport.pdf)).

The central difference between Riak datatypes and other data stored in Riak is that datatypes are **operations based**. Instead of the usual reads, writes, and deletes performed on key/value pairs, you instead perform operations like removing a register from a map, or telling a counter to increment itself by 5, or enabling a flag that was previously disabled (more on each of these types below).

One of the core purposes behind datatypes is to relieve developers using Riak of the burden of producing data convergence at the application level by absorbing some of that complexity into Riak itself. Riak manages this complexity by building eventual consistency _into the datatypes themselves_.

You can still build applications with Riak---and you will always be able to---that treat Riak as a highly available key/value store. That is not going away. What _is_ being provided is additional flexibility and more choices.

The trade-off that datatypes present is that using them takes away your ability to customize how convergence takes place. If your use case demands that you create your own deterministic merge functions, then Riak datatypes might not be a good fit.

## Riak's Datatypes

There is a vast and ever-growing number of CRDTs. Riak implements five of them in total: **flags**, **registers**, **counters**, **sets**, and **maps**. Each will be described in turn in the sections below.

### Flags

Flags behave much like Boolean values, with two possible values: `enable` and `disable`. Flags cannot be used on their own, i.e. a flag cannot be stored in a bucket/key by itself. Instead, flags can only be stored within maps.

#### Operations

Flags support only one two operations: `enable` and `disable`. Flags can be added to or removed from a map, but those operations are on the map and not on the flag directly.

#### Examples

* Whether a tweet has been retweeted
* Whether a user has signed up for a specific pricing plan

### Registers

Registers are essentially named binaries (like strings). Any binary value can act as the value of a register. Like flags, registers cannot be used on their own and must be embedded in Riak maps.

#### Operations

Registers can have the binaries stored in them changed. They can be added to and removed from maps, but those operations take place on the map in which the register is nested and not on the register itself.

#### Examples

* Storing the name `Cassius` in the register `first_name` in a map called `user14325_info`
* Storing the title of a blog post in a map called `2010-03-01_blog_post`

### Counters

Counters are the one Riak datatype that existed prior to version 2.0 (introduced in version 1.4.0). Their value can only be a positive or negative integer. They are useful when a fairly accurate estimate of a quantity is needed, and not reliable if you require unique, ordered IDs (such as UUIDs), because uniqueness cannot be guaranteed.

#### Operations

Counters are subject to two operations, incremement and decrement, whether they are used on their own or in a map.

#### Examples

* The number of people following someone on Twitter
* The number of "likes" on a Facebook post
* The number of points scored by a player in an in-browser role-playing game

### Sets

Sets are basic collections of binary values, such as strings. Sets can be used either on their own or embedded in a map.

#### Operations

They are subject to four basic operations: add an element, remove an element, add multiple elements, or remove multiple elements.

#### Examples

* The UUIDs of a user's friends in a social network application
* The items in an e-commerce shopping cart

### Maps

Maps are the richest of the Riak datatypes because within the **fields** of a map you can nest _any_ of the five datatypes, including maps themselves (you can even embed maps within maps, and maps within those maps, and so on).

#### Operations

You can perform two types of operations on maps:

1. Operations performed directly on the map itself, which includes adding and removing fields from the map (e.g. adding a flag or removing a counter).
2. Operations performed on the datatypes nested in the map (e.g. incrementing a counter in the map or setting a flag to `enable`). Those operations behave just like the operations specific to that datatype.

#### Examples

Maps are best suited to complex, multi-faceted datatypes. The following JSON-inspired pseudocode shows how a tweet might be structured as a map:

```
Map tweet {
    Counter numberOfRetweets,
    Register username,
    Register tweetContent,
    Flag favorited?,
    Map userInfo
}
```

## Riak Datatypes Under the Hood

Conflicts between replicas are inevitable in a distributed system like Riak. If a map is stored in the key `my_map`, for example, it is always possible that the value of `my_map` will be different in nodes A and B. Without using datatypes, that conflict must be resolved using timestamps, vclocks, dotted version vectors, or some other means. With datatypes, conflicts are resolved by Riak itself using a subsystem called [`riak_dt`](https://github.com/basho/riak_dt).

The beauty of datatypes is that Riak "knows" how to resolve value conflicts by applying datatype-specific rules. In general, Riak does this by remembering the **history** of a value and broadcasting that history along with the current value. Riak uses this history to make deterministic judgments about which value is more "true" than the others.

To give an example, think of a set stored in the key `my_set`. On one node, the set has two elements (A and B), and in another node the set has three elements (A, B, and C). In this case, Riak would choose the set with more elements, and then tell the former set: "This is the correct value. You need to have elements A, B, and C." Once this operation is complete, the set will have elements A, B, and C on both nodes.

And so conflicts between replicas of sets will always weighted, so to speak, in favor of sets with more elements. All datatypes have their own internal weights that dictate what happens in case of a conflict, as outlined in the table below:

Datatype | General rule
:--------|:------------
Flags | `enable` wins over `disable`
Registers | The most chronologically recent value wins, based on timestamps
Counters | 
Sets | 
Maps |

In a production Riak cluster being hit by lots and lots of writes, value conflicts are inevitable, and Riak datatypes are not perfect, particularly in that they do _not_ guarantee [[strong consistency]]. But the rules that dictate their behavior were carefully chosen to minimize the downsides associated with value conflicts.

<div class="note">
<div class="title">Note</div>
Conflicts between values very often exist for only a few milliseconds, and so eventual consistency should be thought of in terms of that timescale (as opposed to minutes or hours).
</div>

## Riak With and Without Datatypes

When using Riak without datatypes, you can still reason in terms of some values between more "true" than others, but only on the basis of objects' metadata, like timestamps or vclocks or dotted version vectors (as in the case of Riak's [[strong consistency]] subsystem), and not on the basis of _the actual content of those values_. With datatypes, it matters to Riak how many elements are in a set and what those elements look like, whether a flag is enabled or disabled, etc. And so the bases of judgment about data are vastly expanded with Riak datatypes.