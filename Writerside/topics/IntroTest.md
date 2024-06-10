# IntroTest

## Introduction

The current architecture of the AppMediator is designed in that way, that a consumer (here: VEDA Horizon)
needs to provide an endpoint to receive an information that something
happened (providing an event type and an object id/key).
The consumer then needs to handle that event probably contacting the AppMediator on its own to
retrieve information for that event/object id.

This kind of architecture is not a “standard“ way in the world of message queues and message brokers.

> To ensure, that our consumer (VEDA Horizon) does not 
> get overloaded by events received, we introduced an 
> asynchronous processing limiting the work in progress and 
> to avoid a synchronous coupling of external systems.
{style='warning'}

To prevent that events get lost we are using an internal persistent event queue.

## Overview
![](Без_названия.png)

## Details

### Event receiving

If the AppMediator calls our endpoint a new thread is created (as always for every request). 
The event is stored synchronously in our event queue and if everything went fine we send a successful response to the caller. 
Due to this approach the event receiving operation workload is not limited by us.

### Event processing

The event processing is done asynchronously using one thread to ensure that all events are processed in the order of how we received them and that the workload is limited. 
To ensure that, the events are sorted by a timestamp and a column which contains a call counter per day (callNumber). 
This was done because the timestamp couldn’t be used to generate values in a unique way (not precise enough even with ms)
and it was easier to manage the counter on our own rather than using and maintaining database sequences (supporting 3 DBMS).

We introduced Spring Retry to try again if an exception occurred.

Due to the fact, that we are responsible for our event processing internally, it is possible to enhance the Retry 
functionality / strategy e.g. based on own exception types inside VEDA Horizon without the need to implement that 
strategy fully in the API layer.

>The execution is done in one transaction for all handlers as we are one application and it would increase complexity to manage a state per handler. 
> If one handler fails the whole transaction fails.
> {style='warning'}

Handlers need to be implemented carefully and you should consider implementing own use case specific error handling 
and rollback mechanisms if needed (Especially when leaving the transaction scope using e.g. asynchronous operations). 
This is not only related to this event handling stuff but also for every other place in the application.

### Errors
If all retry operations failed it is a permanent error which is logged to the log file (`AppMediatorEventProcessor`, state `ERROR`)
and into the database including a stack trace (`ho_appmediator_event`, `ho_state=ERROR`, `ho_error_details`).