---
title:  "11 Gradual Methods On How To Scale A Database"
date:   2020-07-13 09:00:00
permalink: how-to-scale-database
---

I used to work in a web shop / app agency and now as a full stack app and web developer. My work’s focus has always been about churning out application fast and furious. There is not much investment, budget and love to shower upon optimization, security and, in particular, scaling concerns.

With some free time on my hand, I decided to look at the topic that has been bothering me since my first ever Ruby On Rails application: database scaling.

The methods to scale a database can be split into 3 categories.

- Application and code
- Database design and schema
- Database infrastructure

## 11. Obliterate N + 1 Queries

This is a pcommon problem](https://secure.phabricator.com/book/phabcontrib/article/n_plus_one/) that can overwork the database with unnecessary number of queries. It generally lacked the use of JOINs in the queries to prefetch data that will be used eventually in the same request.

Common problem, common solution. Use JOINs to eager load the data beforehand to spare your server multiple trips to the database.

As an application level optimization on the code base, it should be a basic practice for backend developers.

I myself, however, do not practise this all the time. As a word of justice to explain myself and probably a fair number of my fellow confrère, we don’t optimize right away because we have clients who don’t really know what they want and the project requirements were not that all that clear to start with. It just does not make financial sense to spend resources optimizing the project.

From a developer’s point of view and experience building up applications from scratch, the priority is functionality, low budget and speed. Optimization is a bonus we do not paid

## 10. Optimize Data Type

Data type optimization is an optimization done on the database schema.

Optimize the space needed for a column. An email should not need the full VARCHAR(255) of a typical string data type for example.

Optimizing the space that each column in each table takes will reduce the time taken for the database to get the required data as it will “traverse less bytes”.

## 9. Normalization

Normalization is an optimization done on the database design and schema.

Split out common but less accessed data into separate tables so that there is less computation required when reading the key tables. This is a form of enumeration at the database level. It keeps reading data efficient and thus reduces the load on the database.

However, be careful not to over normalized your tables or it will require a lot of unnecessary JOINs that can quickly bloat up computational needs. An example that I have came across is the normalization of ‘days’ into a table of permanently 7 rows and 2 columns.

# 8. Indexing

Indexing is an optimization done in the database design.

Indexing allows the database to look through a mapping table to find the required row of data in the corresponding table rather than the whole table of data itself.

A mapping table is much lightweight, hence it reduces the time needed to get to a data, freeing up more resources for the database to handle more incoming requests.

Think of it as the table of content in a lengthy web page, book or catalog of grocery. You would do a “control F” to look for the information you want via the table of content rather than read from start to finish until you get to your data. That is indexing essentially.

In the realm of database indexing, there is also multiple columns indexing where multiple columns are indexed instead of one. This is useful for queries involving either 1 or all of those columns. This same multiple column index can be use for queries on a single column too, which saves space from redundant individual single column indices, and reduce computation of writes to the database to update more indexes than necessary.

Another method related to indexing is to use index prefixing. You would index only the front part of a column in this case. This reduces the space required for the index table, and a cleverly thought index prefixing architecture can boost performance substantially, especially when dealing with much larger data.

Sometimes, you may need to use FORCE INDEX or USE INDEX to get the database to play nicely with the indices you have created and the query.

A tip when debugging indexing in your database: use EXPLAIN command to help you debug what your index is doing to make sure it working the way you think it should.

Credits to this SQL talk by Byrana Knight in RailsConf 2017, here are some ways to use indices, that is not following the best practices, to boost performance.

https://youtu.be/BuDWWadCqIw

##  7. Database Views

Database views is an optimization done on both application and database design level.

Database views are stored queries in the database. They can store temporary results and have an index attached to them.

The advantage of using database views over only indexing your tables is that the database now only has to go through the filtered results from the SQL query in the database view, as compared to an index which consist of all results without the filtered.

For example, if a database view has a query that only looks at records for this year, then your database will only be searching the records for only this year. Using only an index, it will have to search through the records since the start of time till this year, which is a lot less efficient. This [stackoverflow answer](https://stackoverflow.com/questions/439056/is-a-view-faster-than-a-simple-query/439061#439061) answers it better.

## 6. Caching

Caching is an optimization done on the database infrastructure level.

Some of the information we display on our websites and apps are derived data from our database. Derived data are raw data from the database that are computed within your application based on business logic. Some examples I can think of are tabulating the total spent by users from an online shop which involves calculating the individual prices of each item, the quantity bought, discounts and miscellaneous fees like taxes and shipping.

These data would not change, given the same raw data and same computing algorithm. So rather than running through the same requests to get the same raw data from the database, and going through the same algorithm, it may make sense to cache it. We typically use cache servers, are not part of the typical databases and are add ons to the infrastructure, to handle this.

Cache servers like [Redis](https://redis.io/) stores data in [RAM](https://en.wikipedia.org/wiki/Random-access_memory) and not on the hard disk like typical databases. The significance of this distinction is that these memory on RAM can be accessed much quickly than those on the hard disk. It is also this exact reason that RAM is much more expensive than memory in the hard disk, thus destroying any idea you might have about using purely cache as the database.

This gives you much to think about what data should be cached and what should not. The art of using cache efficiently to break up the bottlenecks in your application requires much experience and experiment.

You also have to use them wisely because most cache have a limit on how much data you can store in it. Redis, for example, as a key-value store, at the very basic level, has a [limit of 512MB for each value](https://redis.io/topics/data-types/#:~:text=Redis%20Strings%20are%20binary%20safe,max%20512%20Megabytes%20in%20length.).

On top of that, you need to be smart about when and how to auto expire and explicitly expire cached data so that they show the latest data according to your application needs. For example, when there is a change in the raw data in your database, the computation has to be done again since we are talking about new and different input values.

## 5. Read Replicas

The use of read replicas is an optimization done on the database infrastructure level.

Read replicas involves spinning up more database copies of the master database to handle read loads. This spreads the load up, leaving mainly the write requests to the master database.

Some read requests that required strong consistency still need to go through the master database. This is due to the latency of data propagation from the master database to these read replicas when there are new changes made to the master database.

If your application is write intensive, this may not be the best tool for the job and it will achieve little improvement in performance.

## 4. Vertical Scaling

Vertical scaling is an optimization done on the database infrastructure level.

This is the oldest trick in the book: throw money at the problem. Upgrade the database or opt for a more [IOPS intensive storage type](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html#USER_PIOPS).

INSERT MEME

This is ultimately a mere stop gap solution as there is a limit on how far this can take us. It is also a costly upgrade for a non future proof solution.

I perceive its main advantage as simply buying us time to prepare for the next level of scaling.

## 3. Vertical Partitioning

Vertical partitioning is an optimization done in the database infrastructure level.

Disclaimer: I have never experienced doing this, but I believe this is what vertical partitioning is theoretically about and loved to be pointed out if I am wrong about it.

This step is slightly different from what most people perceive of sharding, which is more commonly horizontal sharding that we will cover later. Vertical partitioning is a form of sharding that is easier to implement. I deem it the appetizer for sharding.

It involves splitting columns and even tables into a separate databases or “shards”. This reduces the data in the main database and thus its computing load. It also spreads the traffic, in particular write requests that replicas are not able to solve, to other shards.

Each shard itself can have its own read replica clusters to further reduce the distribute the load.

However, this complexity will seep into the application level as now your application needs to know which database to connect to to write or read whatever data.

## 2. Hybrid Databases

Vertical partitioning is an optimization done on the database infrastructure level.

Disclaimer: I have never experienced doing this, but I believe this is what vertical partitioning is theoretically and loved to be pointed out if I am wrong about it.

This is a follow up on vertical partitioning. We can use new and more appropriate technologies to the new shards that can manage that part of the application better.

For example, we can use NoSQL databases to handle the historical coordinates of vehicles for a location tracking module. The requirements of this module is places itself more towards the availability and partition tolerance in the [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem). There is no imperative need for [ACID](https://en.wikipedia.org/wiki/ACID) properties to be upheld in database transactions, and [eventual consistency under the principles of BASE](https://en.wikipedia.org/wiki/Eventual_consistency) is sufficient for this module, on a very general level. This allow us to utilize the scaling capabilities of the NoSQL database, at the expense of consistency in the data, which is something we can deal with.

That said, not all NoSQL databases are made equal. They do not all sacrifice consistency for availability and scaling. There are many flavors of NoSQL that will fit different requirement of your module and it is all about finding the correct tool for the correct job.

Another example of using hybrid databases is when you have a highly analytical application. We can partition the tables involved in the data computation into a shard, and perform an [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load) process to store the data in a more efficient structure in databases that are more appropriate for analytical functions, like Amazon Redshift and Google’s BigQuery. It takes away the computational load from your application and database.

The advantage of this partitioning allow you to scale only the bottlenecks of your application in the most cost productive manner.

An example of this vertical partitioning is done by [Airbnb for their chat module](https://medium.com/airbnb-engineering/how-we-partitioned-airbnb-s-main-database-in-two-weeks-55f7e006ff21). They identified it as a bottle neck in their application and acted accordingly to it. They did not use a different technology for that partition in this case.

## 1. Sharding

Sharding, or horizontal partitioning, is an optimization done on the database infrastructure level.

Disclaimer: I have never experienced doing this, but I believe this is what horizontal partitioning is theoretically and loved to be pointed out if I am wrong about it.

Eventually, some of your tables will have so much data that there is a need to split the rows in the tables into different shards. For example, the first 10 million rows will be, in the same table, moved to a shard located in USA, the next 10 million will be, in the same table, placed in another shard located in Germany, and so on.

Usually at this point of time, you will have a handful of clusters of vertical shards. Horizontally sharding each of these clusters will not be manageable. I believe it is a complex mess to be handling this.

This also bring about new problems like cross shard latency at a global scale and application complexity to route the data to the correct shard. Add in the requirement for data recovery it is time to update your resume, as [Mr. Sugu Sougoumarane](https://twitter.com/ssougou?lang=en) mentions below, in his talk about the [Vitess](https://vitess.io/) tool, which will bring me to the next point on this tool as a solution to sharding.

https://youtu.be/q65TleTn2vg

Before you carry out sharding, even for vertical partitions, you may want to consider Vitess. It is a database clustering management system for horizontal scaling to save you the complexity of handling that yourself as you scale so that you can spend your resources on the improving the application itself, which is what ultimately matters.

If I ever get to the point of having to do sharding, at least this will be the first tool that I will research and study more about to tackle the problem.
