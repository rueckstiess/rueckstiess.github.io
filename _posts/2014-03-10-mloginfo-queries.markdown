---
layout: post
title: Query Aggregation with mloginfo
date: 2014-03-20 00:12:02
categories: mtools
---

With the release of mtools 1.1.4, _mloginfo_ receives a new section to aggregate queries in a log file or system.profile collection. The section can be invoked with the `--queries` argument. It aggregates all queries (including the query part of updates), grouped by the query pattern. 

The section shows a table with namespace (in usual `database.collection` syntax), the query pattern, and various statistics, like how often this query pattern was found (count), the minimum and maximum execution time, the mean and the total sum. The list is sorted by total sum (last column), as that reflects the overall work the database has to perform for each query pattern. 

This overview is very useful to know which indexes to create to get the best performance out of a MongoDB environment. Optimization efforts should start at the top of the list and work downwards, to get the highest overall improvement with the least amount of index creation.


Below is an example output of a query section:

```
QUERIES

namespace                    pattern                                        count    min (ms)    max (ms)    mean (ms)    sum (ms)

serverside.scrum_master      {"datetime_used": {"$ne": 1}}                     20       15753       17083        16434      328692
serverside.django_session    {"_id": 1}                                       562         101        1512          317      178168
serverside.user              {"_types": 1, "emails.email": 1}                 804         101        1262          201      162311
local.slaves                 {"_id": 1, "host": 1, "ns": 1}                   131         101        1048          310       40738
serverside.email_alerts      {"_types": 1, "email": 1, "pp_user_id": 1}        13         153       11639         2465       32053
serverside.sign_up           {"_id": 1}                                        77         103         843          269       20761
serverside.user_credits      {"_id": 1}                                         6         204         900          369        2218
serverside.counters          {"_id": 1, "_types": 1}                            8         121         500          263        2111
serverside.auth_sessions     {"session_key": 1}                                 7         111         684          277        1940
serverside.credit_card       {"_id": 1}                                         5         145         764          368        1840
serverside.email_alerts      {"_types": 1, "request_code": 1}                   6         143         459          277        1663
serverside.user              {"_id": 1, "_types": 1}                            5         153         427          320        1601
serverside.user              {"emails.email": 1}                                2         218         422          320         640
serverside.user              {"_id": 1}                                         2         139         278          208         417
serverside.auth_sessions     {"session_endtime": 1, "session_userid": 1}        1         244         244          244         244
serverside.game_level        {"_id": 1}                                         1         104         104          104         104
```

A few comments about the query pattern: Ideally, _mloginfo_ would use exactly the same process of grouping the queries as MongoDB's query optimizer and plan cache does. However, that is a quite complex process, and imitating it would be a lot of work and may still not match in every case. Instead, the query pattern recognizer in mtools takes a simple and predictable approach. It removes all the values and replaces them with a 1 (like indexes are defined). It also reduces `$` operators like `$in`, `$gt`, `$exists` as those are usually handled efficiently with the appropriate index. Operators like `$ne`, `$nin` remain, because they can't be efficiently answered with an index. This does not cover every single case, for example `{$exists: false}` is one of the operators that should probably remain in the pattern, but it works well for most queries.

More information and usage examples for _mloginfo_ (including the other section types `--distinct`, `--connections`, `--restarts`) are available on the [mtools wiki](https://github.com/rueckstiess/mtools/wiki/mloginfo).
 

