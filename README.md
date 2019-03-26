## Process
The solution submitted is to demonstrate the startegy and approach I would follow in order to solve such a problem in real world.

### Design
Being fairly new to Kotlin, I first analyzed the architecture as shown in the read me, and looked at the code structure. Some of the challenges I found are below :
1. Scalability.
2. Concurrency.
3. Reliability.

Below is what I proceeded with for now:
1. Application is supposed to charge invoices simultaneously by multiple threads. In the real world Payment Provider would most likely be a regular REST service, so we can use some nonblocking http client to handle multiple requests by a single thread
2. I decided to use database locking (`SELECT..FOR UPDATE`) for concurrency, and to keep database transactions short I introduced new invoice type enum(`IN PROGRESS`) so the thread that fetches invoices first locks rows, changes it's statuse to `IN_PROGRESS` and then commit a transaction.

### Billing Service
I came across Project Reactor during research. It allows to easily wrap synchronous, blocking calls and separate them from rest of the code. Later we could replace blocking calls with nonblocking implementation like [reactor netty client](https://github.com/reactor/reactor-netty)
and [r2dbc](https://github.com/r2dbc).

First, I created a pending invoices publisher. 
It works in pull manner i.e. it publishes next elements only when a subscriber request them (`sink.onRequest`). 
Then I focused on charging a single invoice and handling all corner cases. It runs on [elastic scheduler](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html#elastic--)
that is a good choice for I/O blocking work.
I added a timeout of 5s (configurable) and added a retry policy similar to network exception using exponential backoff strategy. 
I assumed our external payment provider request is safe to be called multiple times with the same invoice - Otherwise, they would have provided `status` method or something similar.
Finally, I connected both parts in `chargeAll` method. The `limitRate` operator controls the number of rows fetched by pending invoices publisher, while `flatMap` enables concurrency (number of concurrent `charge` processes limited to 50 not to bring down the external payment service).

### Scheduling 
I run invoice processing tasks one by one with 10 minutes delays if today is the 1st day of a month. I am assuming that new pending invoices could appear in any moment.

## Alternative approach
Based on my research, [Quartz Scheduler](http://www.quartz-scheduler.org) with persistent Job Store could be used for scheduling as well.

## Remarks
Please note that this is not a production ready solution as I have not been able to test each module thoroughly. My intention here was to be able to demonstrate my thought process and the approach I would take to build such a solution.
