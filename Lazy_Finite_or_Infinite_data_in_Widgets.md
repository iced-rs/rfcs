# Lazy Finite or Infinite data in Widgets.

## Summary

A `DynamicData` Struc linked to a `DataLoader` than can fetch more data/trim its content based
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

Seeing as 2. usually leads to 1. I propose a Struct that
updates (grows, trims) an inner `data` field when requested. This `data` can
subsequently be turned into Element(s) during the view() method call of the
application.


## Detailed Explanation


A `DynamicData` Struc linked to a `DataLoader` than can fetch more data/trim its content based
on received messages `FetchPrevious(how_many)`, `FetchNext(how_many)` / an update
 call.


Has a mutable buffer_size to let us choose how much to keep:
* when at maximum capacity, `FetchPrevious(n)` pops `n` end values and inserts `n`
new values at the start
* when at maximum capacity, `FetchNext(n)` pops `n` start values and appends `n`
new values at the end.

Has an `OrderBy` trait that forces the Developper to define how to fetch more
items (basically to now what **previous**, and **next** items are).


The `DynamicData` struct exposes methods to change its capacity, its current
position in the data, to ask the DataLoader to get more data in either direction.

Since all these methods are public, the application can simply call them for any
reason, for instance if it receives a button_pressed message. 
