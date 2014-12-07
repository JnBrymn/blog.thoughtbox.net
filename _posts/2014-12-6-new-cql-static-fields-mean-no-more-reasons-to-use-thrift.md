---
layout: post
title: New CQL Static Fields Mean No More Reasons to Use Thrift
---
The Cassandra Query Language has largely eclipsed Thrift as the dominant interface to Cassandra, but until recently there was still one nagging little hole in CQL functionality -- something that was easy to do with Thrift but impossible with CQL! Let's say that you have a piece of data that is the same for every row across a partition of that data. How can you store that piece of data efficiently? With Thrift it's pretty simple, but until recently the answer with CQL was "you can't!". Finally, with the introduction of *static fields* in Cassandra 2.0.6 this longstanding problem has been resolved! Let's dig in and see what's changed.


## Where We've Come From 

CQL hasn't been around that long, but it has so quickly overtaken Thrift that many people, especially new-comers, may be unaware why anyone would have ever wanted to use Thrift with Cassandra in the first place -- so it's time for a review. The Thrift API is minimalistic, it provides the user with a razor thin abstraction over the actual data structures of Cassandra meaning that you can implement your data model absolutely any way you desire. Initially this ability to directly interface Cassandra's data structures was touted as a benefit -- remember when "schemaless" was something people felt like bragging about? But ultimately this freedom came at a great price: complexity. For instance, schemalessness; rarely is data actually without structure, and if you don't have a central place for your schema (here I'm thinking `DESCRIBE TABLE fruits`) then the schema is implicitly dispersed throughout the code that accesses and uses the data. This leads to code that is difficult to understand and maintain. Another problem with standardizing around such a low-level API is that it almost always requires the user to create some sort of client or ORM so they don't have to deal with the low-level headaches every time they modify code. But because of this, soon the Cassandra community found itself flooded with tons of clients that implemented very similar -- though subtly different -- Cassandra data modeling and access patterns.

At this point the need to consolidate the community around a higher level abstraction had become obvious and the community answered with CQL, a SQL-like language that makes it a lot, lot, lot easier to do what you want to do with Cassandra about 99% of the time. It standardizes and implements best practice data modeling and access patterns, it's protocol effectively consolidated the fragmented client ecosystem, it reintroduces the idea of schema, and a whole ton of other delightful things. But it's the 1% that sometimes bothers me, because if you're in that 1% it's often impossible to use CQL to do what Thrift could. And one of the big holdouts that I had been waiting for is effectively taken care of with the introduction of static fields. Static fields allow the user to *efficiently* associate a single piece of data with an entire partition of rows. For instance, you can do things like this:

```mysql
// make table to track availability of products at several store locations
> CREATE TABLE inventory (
      sku text,
      store_id int,
      prod_name text static,
      PRIMARY KEY (sku,store_id)
);

// most importantly SuperSoda which is available at store 56
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

But remember, you could do this with Thrift all along! But to understand how we need to pull up the hood and get an quick review of Cassandra's internal data structure.

## Underlying Cassandra Data Structure

When you really want to take advantage of Cassandra's unique performance characeteristics it's important to understand the mapping between CQL and Cassandra's internal data structures. If you're interested in digging in, you should check out my Cassandra Community Webinar ["Understanding How CQL3 Maps to Cassandra's Internal Data Structure"](https://www.youtube.com/watch?v=UP74jC1kM3w)), but for now let's have a quick review.

Let's say you make a simple table in CQL:

```mysql
> CREATE TABLE inventory (
      sku text,
      store_id int,
      price decimal,
      PRIMARY KEY (sku,store_id)
);

```

If you inserted data into this table then it might look something like this:

```mysql
> SELECT * FROM inventory ;

 sku    | store_id | price
--------+----------+-------
 111def |       42 |  4.35
 111def |       78 |  4.25
 111def |       90 |  4.39
 123abc |       56 |  1.97
 123abc |       78 |  1.93
 123abc |       90 |  1.69
```

But inside of Cassandra there are actually only two "long-rows" which have now come to be called partitions. These partitions look something like this:

```
  ROW   | 42:price | 78:price | 90:price |
  KEY   +----------+----------+----------|
 111def | 4.35     | 4.25     | 4.39     |
 
  ROW   | 56:price | 78:price | 90:price |
  KEY   +----------+----------+----------|
 123abc | 1.97     | 1.93     | 1.69     |
 
```

Without getting into too many details here, there are just a few important things to notice:
* The internal row key is just the first part of the CQL primary key.
* The START HERE


While in CQL we now speak of tables and rows, the way that Cassandra thinks about the data internally is in terms of a *big table* which has a different notion of rows and columns START HERE

INSERT IMAGE: left side CQL: `describe table`, an example table full of data, right side same table in internal data structure

critical point is how the arrangement works with clustering and partition portions of the primary key

mention link to CQL talk

## Implementing Static Field Behavior with Thrift

## How Static Fields are Actually Implemented



Comment about other neat stuff in CQL with Cass2.1
* user defined types (but frozen)
* tuples
* It will only get better from here (loosening up restrictions on freeze, user-defined types, etc.)

Facts:
* static fields can't be part of a primary key

Question:
* what does frozen do?

TODO: 
* Find good photo for static fields (thrift)
* Add a paragraph before first that has the word "freeze in it and is a better summary of the post"
* rename stuff to "static"
* Make better title

