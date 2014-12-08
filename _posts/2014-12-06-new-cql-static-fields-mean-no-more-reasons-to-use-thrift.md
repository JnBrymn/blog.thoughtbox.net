---
layout: post
title: New CQL Static Fields Mean No More Reasons to Use Thrift
---
The Cassandra Query Language has largely eclipsed Thrift as the dominant interface to Cassandra, but until recently there was still one nagging little hole in CQL functionality -- something that was easy to do with Thrift but impossible with CQL! Let's say that you have a piece of data that is the same for every row across a partition of that data. How can you store that piece of data efficiently? With Thrift it's pretty simple, but until recently the answer with CQL was "you can't!". Finally, with the introduction of *static fields* in Cassandra 2.0.6 this longstanding problem has been resolved! Let's dig in and see what's changed.

<figure>
    <img src='/assets/electrostaticgenerator.jpg' alt='missing' class="centered"/>
</figure>

## Where We've Come From 

CQL hasn't been around that long, but it has so quickly overtaken Thrift that many people, especially new-comers, may be unaware why anyone would have ever wanted to use Thrift with Cassandra in the first place -- so it's time for a review. The Thrift API is minimalistic, it provides the user with a razor thin abstraction over the actual data structures of Cassandra meaning that you can implement your data model absolutely any way you desire. Initially this ability to directly interface Cassandra's data structures was touted as a benefit. (Remember when "schemaless" was something people felt like bragging about?) But ultimately this freedom came at a great price: complexity. For instance, schemalessness -- rarely is data actually without structure, and if you don't have a central place for your schema (here I'm thinking `DESCRIBE TABLE fruits`) then the schema is still there but it's implicitly dispersed throughout the code that accesses and uses the data. This leads to code that is difficult to understand and maintain. Another problem with standardizing around such a low-level API is that it almost always requires the user to create some sort of object mapping layer so they don't have to deal with the low-level headaches every time they modify code. But because of this, the Cassandra community soon found itself flooded with tons of object mapping clients that implemented very similar, *and subtly different*, Cassandra data modeling and access patterns.

At this point the need to consolidate the community around a higher level abstraction had become obvious and the community answered with CQL, a SQL-like language that makes it a lot, lot, lot easier to do what you want to do with Cassandra about 99% of the time. It standardizes and implements best practice data modeling and access patterns, it's protocol effectively consolidated the fragmented client ecosystem, it reintroduces the idea of schema, and CQL provides a whole ton of other delightful things. But it's the 1% that sometimes bothers me, because if you're in that 1% it's often *impossible* to use CQL to do what Thrift could. And one of the big holdouts that I had been waiting for is effectively taken care of with the introduction of static fields. Static fields allow the user to *efficiently* associate a single piece of data with an entire partition of rows. For instance, you can do things like this:

```mysql
// make table to track availability of products at several store locations
> CREATE TABLE inventory (
      sku text,
      store_id int,
      prod_name text static,
      PRIMARY KEY (sku,store_id)
);

// most importantly SuperSoda which is available at store location 56
> INSERT INTO inventory (sku, store_id, prod_name)
      VALUES ('123abc',56,'SuperSoda');

// oh yeah, and SuperSoda is also at stores 78 and 90
> INSERT INTO inventory (sku, store_id)
      VALUES ('123abc',78);
> INSERT INTO inventory (sku, store_id)
      VALUES ('123abc',90);

// time passes ... we forget ...

// where was SuperSoda again?
> SELECT * FROM inventory;

 sku    | store_id | prod_name
--------+----------+-----------
 123abc |       56 | SuperSoda
 123abc |       78 | SuperSoda
 123abc |       90 | SuperSoda
```

The neat thing here is that you don't have to specify the name "SuperSoda" again with every insertion. And why should you? You would expect sku 123abc to always be associated with the same product name.

But this gets much more interesting. Let's say that SuperSoda Co. gets bought out by a multinational corporation that believes sugary carbonated beverages should only be referred to as "pop". An organizational restructuring takes place; meetings are held behind closed doors; and the product once known as SuperSoda emerges with a new name...

```mysql
> UPDATE inventory SET prod_name = 'PowerPop'
      WHERE sku = '123abc';

> SELECT * FROM inventory;

 sku    | store_id | prod_name
--------+----------+-----------
 123abc |       56 | PowerPop
 123abc |       78 | PowerPop
 123abc |       90 | PowerPop
```
See? You only have to update the product name once and it gets applied to the entire partition.

Though you wouldn't necessarily use static fields every day, I think you can see how they would come in handy when you really needed them. And there was no reasonable way to do this before. Prior to static fields you would either have to duplicate the product name for every row by hand (much more to store) or you would have to create a special product metadata table and incur the cost of an extra lookup whenever you needed to find a product name. What a pain!

But remember, you could do this with Thrift all along! And to understand how we need to pull up the hood and get an quick review of Cassandra's internal data structure.

## Underlying CQL and Cassandra Internal Data Structures

When you really want to take advantage of Cassandra's unique performance characteristics it's important to understand the mapping between CQL and Cassandra's internal data structures. If you're interested in digging in, you should check out my Cassandra Community Webinar ["Understanding How CQL3 Maps to Cassandra's Internal Data Structure"](https://www.youtube.com/watch?v=UP74jC1kM3w)), but for now let's have a quick review.

Let's say you make a simple table in CQL:

```mysql
> CREATE TABLE inventory (
      sku text,
      store_id int,
      price decimal,
      amt int,
      PRIMARY KEY (sku,store_id)
);

```

If you inserted data into this table then, queried from CQL, it might look something like this:

```mysql
> SELECT * FROM inventory ;

 sku    | store_id | amt | price
--------+----------+-----+-------
 111def |       42 | 332 |  4.35
 111def |       78 | 315 |  null
 123abc |       56 | 662 |  1.97
 123abc |       78 | 489 |  1.93
```

But inside of Cassandra the data is actually represented in two "long-rows" which have now come to be called partitions. These partitions look something like this:

```
  ROW   | 42:amt | 42:price | 78:amt
  KEY   +--------+----------+--------
 111def |    332 |     4.35 |    315

  ROW   | 56:amt | 56:price | 78:amt | 78:price 
  KEY   +--------+----------+--------+----------
 123abc |    662 |     1.97 |    489 |     1.93 
 
```

Without getting into too many details here, there are just a few important things to notice:

* The internal row key corresponds to the value of the *first part* of the primary key (called the partitioning key). This is why there are only 2 internal rows.
* The internal column names are compound:
  * The first part of the internal column names corresponds to the *remaining part of the primary key* (called the clustering key).
  * The second part of the internal column names is the literal name of the CQL column.
* Internal rows can have any number of columns. The exact number of colums is `(number of unique values of the clustering key) * (the number of CQL fields + 1)` less any fields you've left null. 

The internal representation of the data is a mess isn't it? Why would anyone do that?! Well, even though it might take a little [thinking to wrap your mind around it](https://www.youtube.com/watch?v=UP74jC1kM3w), this convolution of the data leads to some very desirable performance characteristics. Namely, the user can specify the data's grouping via the partitioning key and ordering via the clustering key. Furthermore, the user can expect queries for ranges of data within these groups to be very fast.

When Thrift was the only interface to Cassandra and when you could insert whatever data you wanted for internal row keys, column names, and column values, patterns very similar to this were touted as "best practices". Naturally, there was a good bit of book keeping involved, so the object mapping layer was built on top of the Cassandra Thrift API. In many ways the creation of CQL is really the logical continuation of this, effectively establishing a canonical version of the patterns that had already been established.

## Implementing Static Field Behavior

As we've well established by this point, using the old Thrift API you could do almost anything you wanted with the internal data representation, so implementing something like static fields is easy. My approach would be to find some way to add the static metadata into a partition so that it always sorted at the beginning of the partition. This way the static data would always be in the same place and easy to find, it would be co-located *with* the data it describes and thus easy to access using the normal Cassandra query mechanisms, and it would only ever be in *one place* instead of copied to every item in the partition. The easiest way to accomplish this is to prefix the static field names with a character that lexicographically sorts before the other content. (Let's pretend that '*' satisfies this requirement.) Then the our inventory data would look something like this:

```
  ROW   | *:prod_name | 42:amt | 42:price | 78:amt
  KEY   +-------------+--------+----------+--------
 111def |  ChocoBacon |    332 |     4.35 |    315

  ROW   | *:prod_name | 56:amt | 56:price | 78:amt | 78:price 
  KEY   +-------------+--------+----------+--------+----------
 123abc |    PowerPop |    662 |     1.97 |    489 |     1.93 
 
```

From there, it's not such a big mental leap to understand how you would fit this back into CQL; just make it so that any time the `prod_name` is required for an item, you retrieve it from this special column and present it with every CQL row in that partition - like so:

```mysql
> SELECT * FROM inventory ;

 sku    | store_id | prod_name  | amt | price
--------+----------+------------+-----+-------
 111def |       42 | ChocoBacon | 332 |  4.35
 111def |       78 | ChocoBacon | 315 |  null
 123abc |       56 |   PowerPop | 662 |  1.97
 123abc |       78 |   PowerPop | 489 |  1.93
```

As it turns out, this is exactly how CQL now implements this behavior. Go ahead and see for yourself - just create a simple example like we have here, then startup the old cassandra-cli (in the bin directory) and type `use <your table name>;` followed by `list inventory;`. The values are all hexadecimal, but if you cross your eyes you can see what's going on just fine.

# And We're All Done Here

So that's the story of how the most annoying Thrift holdout finally got patched. But this is only the start of exciting CQL improvements. Perhaps sometime soon I'll blog about the new user defined types and about tuples. Or... maybe I'll just direct you to [Patrick McFadens SlideShare presentation](http://www.slideshare.net/fullscreen/patrickmcfadin/real-data-models-of-silicon-valley) which should give you the gist of all that is new and cool in Cassandra 2.1.
