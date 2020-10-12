---
title: 015 - Avoiding Event Reordering Effects
description: Prevent events from getting reordered inside Jet jobs
---

*Since*: 4.4

## Goal

Remove the effects of event reordering in the Jet Jobs

## Problem statement

Events are getting reordered inside Jet jobs. Thus, Jet only enables the
users to write jobs that are insensitive to the encounter order:
jobs including only stateless mapping and aggregation based on
commutative-associative functions etc. But, this out-of-order affects
the correctness of jobs including state.

Jet execution engine tries to increase data parallelism as much as
possible while sending the events originating from one source DAG node
to multiple DAG nodes. This is why event disordering happens.

![Events Getting Reordered](assets/events_getting_reordered.svg)

After these events are distributed to multiple nodes, the occurrence
order of events is partially disrupted. After these disruption, if these
events encounter an order-sensitive transform(operator), they will cause
unexpected results.

## Design

There are two possible solutions for this problem:

- To prevent event reordering from happening
- To sort the events in the same window before they encounter the
  order-sensitive transforms(operators)

Both approaches have different effects on the performance. To make the
best of both of them, I drafted a design which is rather a hybrid
approach, in which we apply these approaches interchangeably during the
pipeline. We decide which one to use, according to the definition of the
associated part of the pipeline.

The main tradeoff we need to consider when creating this design is
allowing event reordering vs performance.

### Prevention of Event Reordering

When distributing events from a one node to multiple nodes, events get
out-of-order. If we especially avoid this situation that breaks the
order in the pipeline, the order of events will be preserved. But, this
comes with the loss of parallelism: The maximum number of nodes in one
stage is restricted by the predecessor stage (LP of the stage <= LP of
the previous stage). Applying this for the entire pipeline may not
always be feasible. E.g. If any stage of the pipeline contains only a
single node (LP=1), following stages have to contain a single node,
which is a suboptimal utilization of parallelism.

### Sorting Events in the Same Window

Jet already offers windowing support using watermark. We sort the events
in the same window so we can put the events in their initial order.
However, in this approach, sorting events requires an extra computation
and we have to wait for the window to close. This increases the latency
with windowSize/2 on average. The issue of sparse events can also occur
in this approach.

To use this solution, we need a more precise timestamp or mark like a
SequenceId - Our timestamp precision is low, resulting in overlapping
events with the same timestamp.

### Smart Job Planning

If we classify the transforms as order-sensitive and order-insensitive,
we can determine whether we will require the initial event order at the
stage of the pipeline. In other words, grouping the transforms according
to whether the result of the transforms is affected by the event order
or not is one of the most important factor that will help us hide the
ordering effects. To clarify why this would be useful, consider this
example case: Suppose the pipeline contains all order-insensitive
transforms. Then, if we are aware that they are order-insensitive, we
can avoid sorting the events unnecessarily. In this way, we make use of
parallelism as much as possible.

Similarly, if we know that a transform explicitly reorder the events, we
can skip any ordering related effort after this transform, even if the
next operation is order-sensitive. It would be unreasonable to sort
events from now on as the order of events after this explicit
order-breaking operation, will already get mixed up. As opposed to this,
some kind of transforms can put the events in a certain order. If there
is no order-sensitive transform before this type of transform, we don't
need to do any ordering related effort until we get here since the
user's explicit order will take effect from now on. These kind of
transforms fix the order that could be previously broken. (e.g. sort)
This group may include aggregation transforms since they make the
effects of reordering disappear (the output of windowed aggregate
transforms are ordered).

With the smart job planning, we determine the subpipelines that we need
to prevent reordering and where we add the sorting stages independently
from the user by considering the classes of transforms. Since we have to
consider the subsequent stages for any stage while making this
determination, it is not enough to visit the pipeline stages only with
topological order.