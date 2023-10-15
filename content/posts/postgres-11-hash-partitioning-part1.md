---
title: "Postgres 11 Hash Partitioning Part1"
date: 2018-10-09T13:40:02-07:00
draft: false
featured_image: '/images/DSCF3884.jpg'
---

# Goal: Explore Postgres 11's Declarative Partitioning features

This entire post is detailed explanation of [a thread](http://www.postgresql-archive.org/Postgres-11-partitioning-with-a-custom-hash-function-tp6048339.html) in postgresql mailing list and wonderful answers by [David Rowley](https://blog.2ndquadrant.com/partition-elimination-postgresql-11/) 

### Why do I need partitions? I already have an index! 

 * A partition is a logical unit that can be
   * `drop`-ed and removed from queried dataset as well as the database
   * `detach`-ed from the frequently queried dataset, but retained in the database for adhoc queries
   * `detach`-ed and `attach`-ed somewhere else (say, another table in same postgres instance)
 * Whenever optmizer decides to fall back to a sequential scan, you would be scanning a smaller table.

### Types of partitioning - `list`, `range`, and `hash`
  * If your dataset needs to be partitioned by date where you can expire data easily, choose `list` or `range`. I will not go into these, because Postgres documentation itself [explains this very well](https://www.postgresql.org/docs/11/static/ddl-partitioning.html#DDL-PARTITIONING-DECLARATIVE). It may be worthwhile to read that document first.
  * `hash` is special, it may be what you need if your key isn't something you can easily enumerate, say a `userid`. `hash` will distribute the data evenly (assuming the `user`s themselves are evenly distributed). Instead of configuring data for `user1` to go into the child table `partition-for-user1`, we would be asking for it to be directed to `partition-N` where `N = hash(key) % M`, M being the number of partitions.
     * Do not use hash partitions if you need to expire data. For example, if you decided to use `HASH(date)`, it wouldn't let you archive a month's worth of data easily. HASH('2018-10-01') has no relation to HASH('2018-10-01')

# Annoyances with builtin `HASH` partitioning {#becausereasons}
* Postgres uses a builtin hash function that isn't one of the popular hashes typically available in programming languages.
* If you need the ability to move partitions around (which was unique to my use case), you need to identity (from your application), which key resolves to which partition.
* See [this thread](https://www.postgresql-archive.org/Hash-Functions-td5961137.html). It seems, it is worthwhile to investigate custom partitioning.

# But first, let us get familiar with the builtin hash partitioning

### Create main table


{{< highlight sql>}}
hyper=> create table foobar (k text, something_else int) partition by hash (k);
CREATE TABLE
{{< / highlight >}}


### Let us try to insert some data
	
{{< highlight sql>}}
hyper=> insert into foobar values ('Hello', 25);
ERROR:  no partition of relation "foobar" found for row
DETAIL:  Partition key of the failing row contains (k) = (Hello).
{{< / highlight >}}

Well, we told postgres to use HASH to partition, but didn't tell it how to resolve a hash value to a partition. Let us see what is the hash for key = 'hello'...

{{< highlight sql>}}
hyper=> select hashtext('Hello');
 hashtext
-----------
 820977155
(1 row)
{{< / highlight >}}
	
NOTE: hashtext() is not really the one PG uses, but that is the closest approximation for now.

### How does `820977155` map to a child table / partition ?

The trick is to do a `mod` using the number of partitions (let us say `4` for now)
	
{{< highlight sql>}}
hyper=> select 820977155 % 4;
 ?column?
----------
        3
(1 row)

hyper=> create table foobar3 partition of foobar for values with (modulus 4, remainder 3);
CREATE TABLE
{{< / highlight >}}
	
This asks PG to take a hash value like `820977155` and do `modulo 4` and find the remainder and use that as the partition identifier, which, `for value=3` is tied to `foobar3`

{{< highlight sql>}}
hyper=> create table foobar3 partition of foobar for values with (modulus 4, remainder 3);
CREATE TABLE
hyper=> \d foobar
                   Table "public.foobar"
     Column     |  Type   | Collation | Nullable | Default
----------------+---------+-----------+----------+---------
 k              | text    |           |          |
 something_else | integer |           |          |
Partition key: HASH (k)
Number of partitions: 1 (Use \d+ to list them.)
	
hyper=> \d+ foobar
                                       Table "public.foobar"
     Column     |  Type   | Collation | Nullable | Default | Storage  | Stats target | Description
----------------+---------+-----------+----------+---------+----------+--------------+-------------
 k              | text    |           |          |         | extended |              |
 something_else | integer |           |          |         | plain    |              |
Partition key: HASH (k)
Partitions: foobar3 FOR VALUES WITH (modulus 4, remainder 3)
{{< / highlight >}}
	
	
### Try again?

{{< highlight sql>}}
hyper=> insert into foobar values ('Hello', 25);
ERROR:  no partition of relation "foobar" found for row
DETAIL:  Partition key of the failing row contains (k) = (Hello).
{{< / highlight >}}
	
Nope we still don't know where `Hello` falls. But this is just an exercise - do I really care which partition number is used?

### Complete defining all partitions (programmatically, if number of partitions is large)

{{< highlight sql>}}
hyper=> create table foobar2 partition of foobar for values with (modulus 4, remainder 1);
CREATE TABLE
hyper=> create table foobar3 partition of foobar for values with (modulus 4, remainder 2);
CREATE TABLE
hyper=> create table foobar4 partition of foobar for values with (modulus 4, remainder 3);
CREATE TABLE
hyper=> \d+ foobar
                                       Table "public.foobar"
     Column     |  Type   | Collation | Nullable | Default | Storage  | Stats target | Description
----------------+---------+-----------+----------+---------+----------+--------------+-------------
 k              | text    |           |          |         | extended |              |
 something_else | integer |           |          |         | plain    |              |
Partition key: HASH (k)
Partitions: foobar1 FOR VALUES WITH (modulus 4, remainder 0),
            foobar2 FOR VALUES WITH (modulus 4, remainder 1),
            foobar3 FOR VALUES WITH (modulus 4, remainder 2),
            foobar4 FOR VALUES WITH (modulus 4, remainder 3)
{{< / highlight >}}

### Try again?
	
{{< highlight sql>}}
hyper=> create table foobar2 partition of foobar for values with (modulus 4, remainder 2);
CREATE TABLE
hyper=> create table foobar1 partition of foobar for values with (modulus 4, remainder 1);
CREATE TABLE
hyper=> create table foobar0 partition of foobar for values with (modulus 4, remainder 0);
CREATE TABLE
hyper=> insert into foobar values ('Hello', 25);
INSERT 0 1
hyper=> select * from foobar;
   k   | something_else
-------+----------------
 Hello |             25
(1 row)
{{< / highlight >}}

### Where did the row land?
	
Instead of querying each of the 4 tables, here is a neat trick

{{< highlight sql>}}
hyper=> select tableoid::regclass, * from foobar;
 tableoid |   k   | something_else
----------+-------+----------------
 foobar1  | Hello |             25
(1 row)
{{< / highlight >}}


# The Impatient Gopher

...that I am, I immediately ported the postgres hash library in C to Go. Although it is evident from the example above that there is more going on there. The library is [here](https://github.com/harikb/pghash), but it turned out to be a waste of time. The library is still useful if you are trying to follow `hashtext()` but not for hash partitionining, as per the [David Rowley](http://www.postgresql-archive.org/Postgres-11-partitioning-with-a-custom-hash-function-tp6048339p6048643.html), there is hash seed involved.


# Looking into using a custom hash function (C extension)

_In case you forgot why I am going this route, see [this]({{<relref "#becausereasons">}})._

Let us declare a postgres function that will use [xxhash](https://github.com/Cyan4973/xxHash)


{{< highlight c>}}
#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"
#include "xxhash.h"
	
PG_MODULE_MAGIC;
	
PG_FUNCTION_INFO_V1(loopyhash);
Datum
loopyhash(PG_FUNCTION_ARGS)
{
    text *arg1 = PG_GETARG_TEXT_PP(0);
    int32 arg1_size = VARSIZE_ANY_EXHDR(arg1);
	
    int64 arg2 = PG_GETARG_INT64(1); // For now, seed is unused, set to zero
	
    unsigned long long retval64 = XXH64(VARDATA_ANY(arg1), arg1_size, 0);
	
    PG_RETURN_INT64(retval64);
}
{{< / highlight >}}


Notice that a `seed` is passed into the hash function, but I ignore it for now. I don't need a seed.

#### Compile and install

{{< highlight bash>}}
$ gcc -fPIC -I/usr/include/postgresql/11/server -c pge0.c && gcc -shared -o pge.so pge0.o
{{< / highlight >}}


### Install extension in to PG.

{{< highlight bash>}}
hyper=# CREATE FUNCTION loopyhash(text, int8) RETURNS bigint
     AS '/tmp/pge/pge.so', 'loopyhash'
     LANGUAGE C STRICT IMMUTABLE;
CREATE FUNCTION
{{< / highlight >}}

### Create operator clas
	
{{< highlight sql>}}
hyper=# create operator class loopyhash_text_ops
for type text
using hash as
operator 1 =,
function 2 loopyhash(text, int8);
CREATE OPERATOR CLASS
{{< / highlight >}}

### Create the table with HASH partitioning, but using the custom operator-class inside `HASH (k...)` expression	
	
{{< highlight sql>}}
hyper=# create table foobar (k text primary key, something_else int) partition by hash (k loopyhash_text_ops);
CREATE TABLE
{{< / highlight >}}

Notice how the expression is `partition by hash (k loopyhash_text_ops)` instead of `partition by hash (k)`

### Define hash-functions and operator class for other data-types, if necessary

Not applicable to my use-case. Skipping it. I am just dealing with a single text column.

### Define partitions

{{< highlight sql>}}
hyper=> \d foobar
                   Table "public.foobar"
     Column     |  Type   | Collation | Nullable | Default
----------------+---------+-----------+----------+---------
 k              | text    |           |          |
 something_else | integer |           |          |
Partition key: HASH (k loopyhash_text_ops)
Number of partitions: 0
	
hyper=> \i foobar.sql
BEGIN
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
COMMIT
hyper=> \d foobar
                   Table "public.foobar"
     Column     |  Type   | Collation | Nullable | Default
----------------+---------+-----------+----------+---------
 k              | text    |           |          |
 something_else | integer |           |          |
Partition key: HASH (k loopyhash_text_ops)
Number of partitions: 4 (Use \d+ to list them.)
	
hyper=> \d+ foobar
                                       Table "public.foobar"
     Column     |  Type   | Collation | Nullable | Default | Storage  | Stats target | Description
----------------+---------+-----------+----------+---------+----------+--------------+-------------
 k              | text    |           |          |         | extended |              |
 something_else | integer |           |          |         | plain    |              |
Partition key: HASH (k loopyhash_text_ops)
Partitions: foobar_00 FOR VALUES WITH (modulus 4, remainder 0),
            foobar_01 FOR VALUES WITH (modulus 4, remainder 1),
            foobar_02 FOR VALUES WITH (modulus 4, remainder 2),
            foobar_03 FOR VALUES WITH (modulus 4, remainder 3)
{{< / highlight >}}
	            

### Try insert

{{< highlight sql>}}
hyper=> insert into foobar values ('Hello', 25);
INSERT 0 1
	
hyper=> select loopyhash('Hello'::text, 0);
     loopyhash
--------------------
 753694413698530628
(1 row)
	
hyper=> select 753694413698530628 % 4;
 ?column?
----------
        0
(1 row)
	
hyper=> select tableoid::regclass, * from foobar;
 tableoid  |   k   | something_else
-----------+-------+----------------
 foobar_03 | Hello |             25
(1 row)
{{< / highlight >}}
	
	
Clearly, we are still off. Even though we ignored the `seed` and did our own calculation, it didn't match with PG.

### Real problem

[Turns out](http://www.postgresql-archive.org/Postgres-11-partitioning-with-a-custom-hash-function-tp6048339p6048757.html), postgresql [applies a method](https://github.com/postgres/postgres/blob/29c94e03c7d05d2b29afa1de32795ce178531246/src/backend/partitioning/partbounds.c#L2075)  `hash_combine64()` on each of the hashes for a multi-column hash expression. In this case it doesn't need to do that, but the code is generic.

I can solve this two ways. Either include this [`hash_combine64()`](https://github.com/postgres/postgres/blob/29c94e03c7d05d2b29afa1de32795ce178531246/src/include/utils/hashutils.h#L32) in my client code or offset for this inside the postgres custom hash function. I chose the latter just so that I can use the files I already created for a prototype. This isn't perfect, and probably has [integer underflow](https://play.golang.org/p/4n3yfUavh4f) issues you may have to [work around](https://play.golang.org/p/uFapke_kNt1). But we are OK in C at least as far as I know. Moreover, we are only interested in the last few bits of the hash (for the modulo operation)

	
{{< highlight c>}}
$ cat pge1.c
#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"
#include "xxhash.h"
	
PG_MODULE_MAGIC;
	
PG_FUNCTION_INFO_V1(loopyhash);
Datum
loopyhash(PG_FUNCTION_ARGS)
{
    text *arg1 = PG_GETARG_TEXT_PP(0);
    int32 arg1_size = VARSIZE_ANY_EXHDR(arg1);
	
    int64 arg2 = PG_GETARG_INT64(1); // For now, seed is unused, set to zero
	
    unsigned long long retval64 = XXH64(VARDATA_ANY(arg1), arg1_size, 0);
	
    retval64 -= 0x49a0f4dd15e5a8e3;
	
    PG_RETURN_INT64(retval64);
}
	
$ rm -fv pge.so && gcc -fPIC -I/usr/include/postgresql/11/server -c pge1.c && gcc -shared -o pge.so pge1.o
removed 'pge.so'
{{< / highlight >}}

### Notice something strange below?

If you query the existing data immediately after updating the hash function...

{{< highlight sql>}}
$ psql -U hyper hyper
psql (11beta4 (Ubuntu 11~beta4-1.pgdg18.04+1))
Type "help" for help.
	
hyper=> select tableoid::regclass, * from foobar;
 tableoid  |   k   | something_else
-----------+-------+----------------
 foobar_03 | Hello |             25
(1 row)
	
hyper=> select * from foobar where k='Hello'::text;
 k | something_else
---+----------------
(0 rows)
{{< / highlight >}}
	
Wait!, what happened to the row? Well. We changed the definition of the hash right from underneath postgres. So the existing data is inaccessible!. Truncate and repopulate, please!

### Try again

{{< highlight sql>}}
hyper=> truncate foobar;
TRUNCATE TABLE
hyper=>
hyper=> insert into foobar values ('Hello', 25);
INSERT 0 1
hyper=> select tableoid::regclass, * from foobar;
 tableoid  |   k   | something_else
-----------+-------+----------------
 foobar_00 | Hello |             25
(1 row)
{{< / highlight >}}
	
### How to verify the hash is really corrected, since we can't call `loophash` to check (it has just been fixed with the offset)?

Add debugging logs
	
{{< highlight c>}}
... 
unsigned long long retval64 = XXH64(VARDATA_ANY(arg1), arg1_size, 0);
	
ereport( INFO, ( errcode( ERRCODE_SUCCESSFUL_COMPLETION ), errmsg( "retval64: %llu\n", retval64 )));
	
retval64 -= 0x49a0f4dd15e5a8e3;
	
ereport( INFO, ( errcode( ERRCODE_SUCCESSFUL_COMPLETION ), errmsg( "retval64: %llu\n", retval64 )));
	
....
{{< / highlight >}}



Now see what we get


{{< highlight sql>}}
hyper=> select * from foobar where k='Hello'::text;
INFO:  retval64: 753694413698530628
	
INFO:  retval64: 13894928895973315681
	
   k   | something_else
-------+----------------
 Hello |             25
(1 row)
	
hyper=> select tableoid::regclass, * from foobar;
 tableoid  |   k   | something_else
-----------+-------+----------------
 foobar_00 | Hello |             25
(1 row)
	
hyper=> select 753694413698530628 % 4;
 ?column?
----------
        0
(1 row)
{{< / highlight >}}

So the real *XX64* of `Hello` is **753694413698530628**, but we returned **13894928895973315681** to postgres so it would do `hash_combine64` on it to get back to the number **753694413698530628** and then do the mod. The number **753694413698530628** is ignored as a PG internal detail and can be ignored.

### Long term solution

Although I took the approach of **subtracting 0x49a0f4dd15e5a8e3** from the hash, this isn't portable if the hash_combine64 was any more complicated. The real *production* solution must instead implement `hash_combine64` in the application forward (that is, don't try to do the reverse operation) and consider that as the offical hash value. Some operations like bitshifting can't be undone. We just got lucky this time around with value of `a` in the expression above to be 0.

In my case, I had spent couple of days to pre-split files using regular XXHASH and didn't want to redo it before proceeding with my tests. Hence this hacky approach to patch loopyhash function instead.


### Your custom code & explain

{{< highlight sql>}}
hyper=> explain select * from foobar where k='Hello';
INFO:  retval64: 753694413698530628
	
INFO:  retval64: 13894928895973315681
	
                           QUERY PLAN
-----------------------------------------------------------------
 Append  (cost=0.00..25.91 rows=6 width=36)
   ->  Seq Scan on foobar_00  (cost=0.00..25.88 rows=6 width=36)
         Filter: (k = 'Hello'::text)
(3 rows)
{{< / highlight >}}
	
Notice your code does run at the query **planner** stage!!!


