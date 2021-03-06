---
layout: post
title:  "Hammers are for nails"
date:   2015-07-09 03:12:41 +0800
categories: stack
comments: true
---

[deepo](http://www.deepo.io/) is upgrading our first minimal infrastructure.

### We've Been Lazy

We used to be on Postgresql(RDS) + Rails(Heroku) + Golang(Heroku). It was simple to start with, but grew to be come a sluggish patchwork. The traffic that we deal with isn't huge by any standard, but we attempted to count uniques on some largish tables. We added master-slave replica to separate the writes and reads, created composite indexes on the queried columns, partitioned the table by date range, have background jobs update a summary table out of a star schema, but still we get very slow response time on the unique count queries.

Yes there's [HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog), and we've realise we should tolerate the 2% error. But we learned that the hard way. By now our partitioned slave throws a timeout on simple select, once in a while. I suspect it is because of my tinkering with column constraints without properly handling existing violations. Or it could be that rails migration of databases don't play nicely with master-slave set up. I have to do the same query on the master instance to avoid a timeout.

On top of the intermitten query problems, there was a pending design issue. Our postgresql acted like a poor man's persistent message queue. We have a `pending_page_views` table from which rows are waiting to be processed, then removed and inserted into the `page_views` table. Those extra writes and reads that results from the clumsy pipeline probably doesn't help the responsiveness of the database. It got the job done, and there were no extra technology to integrate. Simply postgres and rails sidekiq. Simple is good right?

In retrospect, it was simple only in terms of coding it out, as we didn't have to climb the learning curve of any other tools. But we were still trying to implement the same thing: persistent buffer queue for data stream processing (we get the time the processing happens too, so we can redo some jobs or even replay the whole thing). The complexity of the process is **the same**. We were not simplifying things, we were just lazy (well, being lazy is only a negative expession of trying to build as
fast as possible with as little effort as possible, which is still a noble thing.)

### SQL vs NoSQL

Along the way a couple of different experienced opinions sugguested we utilize a nosql database to take care of one part of the app, where writes is intensive, and queries are never joined.

"But, you mean I'll have to rewrite all my queries?!" Yes.

Luckily for us, it wasn't that painful at all. The `pending_page_views` and `page_views` aren't really "relational" data anyway.
They are more like [log data](http://docs.mongodb.org/ecosystem/use-cases/storing-log-data/) (who visit what page at what time).

We then have have Postgresql(RDS) + Rails(EC2) + MongoDB(EC2) + Golang(EC2).

We keep app configurations that face client admins as relational data. Postgresql for relational data, MongoDB for non-relational data. Sounds good! Although the complexity is now increased.

### AMQP

I've heard about [rabbitmq](https://www.rabbitmq.com/) before, but didn't really understand why I need it. Because I didn't. I probably still don't understand it. But I try: you can make a pub/sub thing, so we can queue and distribute the processing of incoming data stream to multile workers. Should fit the bill for our batch jobs on `pending_page_view`. It's an easy-to-setup AMQP protocal that persists messages;
so if the server shuts down we can just restart and continue picking up jobs. (I finally understood [Apache Kafka](http://kafka.apache.org/) too)

So now: Postgresql(RDS) + Rails (EC2) + MongoDB(EC2) + Golang+Rabbitmq(EC2)

### We need to run Python

As we become more mature on our data-science department, it became a requirement that we are able to run python scripts. We need to recalculate similarity indexes every time a new item enters the pool, and we need to reweight our message distribution weights periodically; these should be threaded. I almost fell into the rabbit hole of trying to let python manage the multiple threads. Instead we restrict python codes to what it does best: data science heavy lifting, and leave the concurrency management to the best guy: [Golang](https://www.youtube.com/watch?v=f6kdp27TYZs)

Finally: Postgresql(RDS) + Rails (EC2) + MongoDB(EC2) + Golang+Rabbitmq+Python(EC2). Much complex! So effort!

### Work in Progress

Some decisions are probably not the best, and some obvious, but it's a starting point for me, questioning "Is this the right tool for this job?"; and everytime I've addressed that, I feel good.
Everyone should feel good, so everyone should ask that question.

However, the answer to "is this the right tool for this job" depends on what's available on your toolbelt.
I've got only a handful and some of them I probably have wrong understandings of (like [AMQP rabbitmq](#)), but I'll be trying to aquire and sharpen more tools.
