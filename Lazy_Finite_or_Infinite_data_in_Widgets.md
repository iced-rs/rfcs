# Lazy Finite or Infinite data in Widgets.

## Summary

A `Dynamic_Data_Wrapper(Vec<Widget>)` than can fetch more data/trim its content based
on received messages `FetchPrevious(how_many)`, `FetchNext(how_many)` / an update
 call.
 
Solves:
  * Both Lazy Finite and Infinite `Scrollable`
  * Both Lazy Finite and Infinite anything that uses a `Vec<Widgets>`
  * Lower computation cost in both cases: Lazyness => smaller widget lists in 
  scope => fewer primitives calculated and simpler/quicker layout calculations. 


## Motivation

Solves:
  * Both Lazy Finite and Infinite `Scrollable`
  * Both Lazy Finite and Infinite anything that uses a `Vec<Widgets>`
  * Lower computation cost in both cases: Lazyness => smaller widget lists in 
  scope => fewer primitives calculated and simpler/quicker layout calculations. 


1. In Iced, large amounts of widget lists may cause performance issues due to the 
nature of update, draw, layout and view methods, even when many widgets are out 
of view, if states change.

2. In Iced, I have not found Infinite list implementation where the 
very large/infinite list is only loaded partially when receiving requests.

Seeing as 2. usually leads to 1. I propose a `Into<Vec<Widget>>` Struct that 
updates (grows, trims) an inner `Vec<Widgets>` when requested. That `Vec<Widget>` is
 used as usual (transparently) by other widgets, like `Columns` and `Rows` for example
 by implementing `Into<Vec<Widget>>` on this wrapper (or From?).


## Detailed Explanation


A `Dynamic_Data_Wrapper(Vec<Widget>)` than can fetch more data/trim its content based
on received messages `FetchPrevious(how_many)`, `FetchNext(how_many)` / an update
 call.

Has a mutable buffer_size to let us choose how much to keep:
* when at maximum capacity, `FetchPrevious(n)` pops `n` end values and inserts `n`
new values at the start
* when at maximum capacity, `FetchNext(n)` pops `n` start values and appends `n`
new values at the end.

Implement `Into<Vec<Widget>>` by returning its current contents.

Has an `OrderBy` trait that forces the Developper to define how to fetch more
items (basically to now what **previous**, and **next** items are).


Systematically appends an additional "Load more" widget (stylable) at the end of the
content which can be triggered to send a message to the DDW to ask it to load more content, for
instance when GUI-interacting on it, or programmaticaly.

Systematically inserts an additional "Load previous" widget (stylable) at the start of the
content which can be triggerd to send a message to the DDW to ask it to load more content, for
instance when GUI-interacting on it, or programmaticaly.

