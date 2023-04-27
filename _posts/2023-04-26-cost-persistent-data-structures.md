---
layout: post
title:  "What's the cost of persistent data structures?"
date:   2023-04-26
tags: [roc, data structures]
mermaid: true
---

Recently in a [Roc](https://roc-lang.org) discussion, a proposal was made to seamlessly convert some Roc data structures to
to [persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure) depending on the use.
Specifically, Roc lists should be convert to a persistent list if they are shared and an update is requested.
In this post, I hope to cover some of the unexpected costs this may have for the Roc language.

## What is a persistent list?

> Note: For this post I am going to be roughly talking about a [RRB vector](https://infoscience.epfl.ch/record/213452/files/rrbvector.pdf).
> It is a high performance persistent list used by some functional programming languages.

Fundamentally, a persistent list is that preserves its previous versions.
This is done so that the list can seem immutable without the cost of duplicating the entire list on every change.

For example:
```coffee
# Create the base list
a = [1, 2, 3, 4]

# Add an element
b = List.append a 5

# As a result, a and b reference the same starter elements: [1, 2, 3, 4],
# but b has extra information that includes the element: 5.
# There was no need to duplicate list a.
```

In reality, performance leads to a more complex picture.
For an RRB vector, the list is stored in a tree structure.

If we create a tree for the above code where each leaf contains 4 elements, it would look like:
<div class="mermaid">
graph TD;
  a-->d1[D1: 1, 2, 3, 4];
  b-->m1[M1: Merge Node];
  m1--0-->d1;
  m1--1-->d2[D2: 5, -, -, -];
</div>

The data structure is a tree to avoid long chains totally destroying the data structure's performance.
In practice, each node will generally contain around 32 elements. This means, having 2 billion elements would only take 7 hops.
As a result, most of the operations are technically `O(log n)`, but with a relatively small `n`.

To show a bit more of how these lists work, here is another code snippet:
```coffee
c =
  List.set b 0 10
  |> List.set 1 11
```

If run after the above snippet, the new graph would be:
<div class="mermaid">
graph TD;
  a-->d1[D1: 1, 2, 3, 4];
  b-->m1[M1: Merge Node];
  m1--0-->d1;
  m1--1-->d2[D2: 5, -, -, -];
  c-->m2[M2: Merge Node];
  m2--0-->d3[D3: 10, 11, 3, 4];
  m2--1-->d2;
</div>

`c` was required to duplicate only node `D1` in order to update it's elements.
It is still able to reference the rest of `b`'s data.

This is what makes persistent data structures extremely helpful.
In theory, you can build up a complex web of interrelated references with little cost.
This removes the (potentially huge) cost of duplication that would be needed otherwise.

## Application to Roc

> Note: For the purposes of the following math, I will assume nodes of 32 elements, which is common in multiple libraries.

In the case of Roc, we are looking at converting to a persistent list as a potential optimization.
If we need to update a list that is not unique, we would convert it to a persistent list before updating.
From that point forward, that list would be a persistent list and no future copies should be needed.
Assuming the list is actually shared and gets mutated, this could avoid lots of repeated duplication.

### Cost of conversion

I think it is very important to note the initial cost of conversion.
An RRB vector is a very specific format and we will not simply be able to implace convert to it.

If we look at a list of 1024 i64s, a persistent list would require `1024 / 32 = 32` leaf nodes.
On top of that, it would need on merge node to group together those 32 nodes.
To preform the conversion, it would take 33 calls to malloc,
32 calls to memcpy of 256 bytes each to random locations, and setting 32 pointers in the merge node.

In comparison, a regular list, would require 1 call to malloc and 1 memcpy of 64kb.
The regular list duplication will always be faster.

### Cost of each use

Even though conversion was expensive, the goal of a persistent list is really to help performance over it's use.
So what are the real costs of using the new list?

Let's start with a simple operation: 
```coffee
List.get data 12
```

This operation does not modify the list, but simply reads from it.
For an persistent list, this will cost 2 pointer lookups (load merge node and load leaf node)
and loading an element from an offset.
For a regular list, this cost 1 pointer lookup (load list) and loading an element from an offset.
So the cost seems to be one extra indirection.

I think it is a bit unfair to say.
In reality, the regular list will likely hit cache while the persistent list is much more likely to have a cache miss.

Here is a simple example:
```coffee
x = List.get data 31
y = List.get data 32
```

For the persistent list, loading both `x` and `y` will lead to a cache miss. They are store in different leaf nodes.
For a regular list, this will never happen.
As such, I would label the cost as even more expensive than just an extra indirection.
It is an extra indirection that is less likely to be cached.

Next, lets turn to mutation. This is were the extra features of the persistent list can actually be used.

Again, let's look at simple operation: 
```coffee
List.set data 12 64
```

Here things get complex. All performance will be very application specific.
It will spefically depend on how large the lists are and how often the lists are shared.

Let's first look at the unique case.
If a persistent list has a unique leaf node, it can simply update the nodes in place.
As such, an `set` is essentially the same cost as `get`. It is 2 pionter lookups and setting an element at an offset.
For a unique regular list on the other hand, the cost of set is 1 pointer lookup and setting an element at an offset.
All of the same caveats of `get` apply here. That said, there is also one extra advantage for persistent lists.
Persistent lists only need the specific leaf node to be unique and not the entire list to be unique.
How often this comes up in practices will be very application dependent.

Next the shared case. For a shared persistent list, there is additional cost of 1 call to malloc and 1 call to memcpy of 256 bytes.
For a shared regular list, there is an additional cost of 1 call to malloc and 1 call to memcpy of the number of bytes in the list.
Clearly, a regular list has a potentially unbounded amount of work while a persistent list has a bounded amount.
In this case, it is likely the persistent list will be faster.

### Other overhead

In all of the above examples, I haad been ignoring book keeping for simplicity.
In reality, every node in a persistent list would need a refcount, a used length, a tag, and maybe some other metadata (extra is used if you want fast `concat`).
Whenever a reference is created or destroyed to a persistent list, it requires traversal of graph of nodes to update refcounts.
All the above peieces will likely add to measurable cost in runtime and memory footprint (read as more cache misses).
These are all extra overhead that the regular list has one instead of many copies of.

### The real cost to Roc

Roc is a pure functional programming language with a focus on performance through oppurtunistic mutation.
In many applications, roc can successful avoid copying lists.
Even when it does copy occasionally, the gains from inplace mutation greatly outweigh the cost of duplication.
I think that adding persistent data structures will reduce the effectiveness of inplace mutation and add extra costs throughout the rest of the application.

#### Reduced effectiveness

Converting to a persistent list is a one way street that many users will never know about.
It is a seamless optimization that happens in the background if the right conditions are met (mutation on a shared list)
Once this conversion has happen, a list, and all of its decendents can never get the benefits from inplace mutation again.

I think in a lot of roc code, this is likely to happen by accident.
There will be 2 references to a list for a short time when there really should have only been one.
Without persistent lists, this would lead to a single duplication, but from that point forward, all optimizations are good.
With the new persistent lists, the full cost of conversion will still happen in these case.
The cost will be much more expensive as mentioned above, and from that point forward, there will be extra costs due to being a persistent list.

I think the real cases in a functional language where a list is shared to many locations and being mutated will be relatively rare.
In those few cases, I think it would be better for a user to note the performance degredation and switch to userland persistent list explictly.

The other case where persistent lists would be useful is when a gigantic list exists that is occassionally accidentally duplicated.
In this case, instead of fixing the real bug that is causing the duplication, the switch to a persistent list can avoid the huge cost of duplication.
Though that is a reasonable solution, I think it is more likely be pointing out bad code than correctly fixing an issue.

#### Extra costs throughout the rest of the application (AKA: Death by a million paper cuts)

Due to persistent lists essentially hijacking where a regular list could have been used, I think there will be tons of extra costs in unexpected locations.

If a list is accidentally turned into persistent list, it will:
- Call malloc all the time because nodes are only 32 elements
- Waste significantly more time toggling refcounts (incrementing a list requires loading every node)
- Have to do multiple random loads to access any data (this also means slower traversal)
- Have to do more bookkeeping for things like rebalancing

#### Other notes

Besides the above costs, persistent lists would not work with seamless slices.
As such, all slices of persistent lists will essentially need to build a new chain of merging nodes.
On top of that, they would need to duplicate the head and tail nodes.
So fast seamless slices will instead be full rbtree contructions.

Persistent lists also would have a cost to convert back into regular lists, so `List.reserve 0` will be expensive.

## Final thoughts

I don't think seamless persistent lists would be a good addition to Roc.
I think they would be an accidental pessimization in many applications.
On top of that, they would make it more confusing for an end user to optimize an app.
They would need to think about uniqueness and inplace updates, seamless slices, and potential persistent lists.

Overall, I think persistent lists should stay an explicit choice in userland instead of being an implicit optimization.