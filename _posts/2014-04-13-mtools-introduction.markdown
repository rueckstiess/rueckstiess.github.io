---
layout: post
title: "Introduction to mtools"
date: 2014-04-13 15:44:02
categories: mtools
---

_mtools_ is a collection of helper scripts, implemented in Python, to parse and filter MongoDB log files (both for mongod and mongos), to visualize information from log files and to quickly set up complex MongoDB test environments on a local machine.

I've started working on _mtools_ about one year ago, when I realized that many of the daily tasks around log files I do as an Engineer at [MongoDB][mongodb] can be automated and scripted. Since then, _mtools_ has grown to a suite of flexible, useful command line tools that are being used by many of our Engineers internally, as well as MongoDB customers and users.

If you find yourself looking at MongoDB log files frequently, then I encourage you to try _mtools_ as well.


### What's in the box?

_mtools_ in its current version 1.1.4 consists of 5 individual scripts: _mloginfo_, _mlogfilter_, _mplotqueries_, _mlogvis_ and _mlaunch_.

* _mloginfo_ should be your first stop on the (potentially) long road of log file analysis. It will parse the file quickly and output general information about its contents, including start and end date and time, line numbers, version (if present in the file) and the type of binary (mongos or mongod). In addition, you can request certain "sections" of additional information; currently those are "queries", "connections", "restarts" and "distinct".

* _mlogfilter_ is all about finding certain events in log files. The script lets you filter on attributes of log messages, like their namespace (database and collection names), their type of operation (queries, inserts, updates, commands, etc.) or by individual connection. You can also search for slow operations above a certain threshold, collection scans (those are the queries not using an index) and other properties. Additional features include slicing the log files by time (from and to with flexible date/time parsing), merging files, shifting them to different time zones or converting timestamp formats, and exporting them to JSON. The key property of _mlogfilter_ is that the output format always remains the same (log lines), so you can pipe the output to another instance of _mlogfilter_, to `grep` or to other scripts like _mplotqueries_. 

* _mplotqueries_ then takes a log file (_mlogfilter_ed or not) and presents the information visually in various ways. A number of different types of graphs are implemented, like scatter plots (showing all operations over time vs. their duration), histograms, event and range plots, and other more specialized graphs like connection churn or replica set changes. Independent of the type of graph, a grouping can be specified to assign different colors to different categories. 

* _mlogvis_ is _mplotqueries_' litte brother, it is very similar in its functionality, but provides a web-based alternative using the [d3.js][d3] javascript visualization engine. This is particularly useful if the dependencies that _mplotqueries_ requires are not installed/available, or if you want to create a self-contained interactive graph that can be sent to customers or colleagues. _mlogvis_ will create a single `.html` file that can be shared, because it loads the d3.js library dynamically.

* _mlaunch_ is a little different to the other scripts, and actually has nothing to do with log file parsing. _mlaunch_'s purpose is to spin up any number of mongodb nodes on your local machine, either as stand-alone, replica sets or sharded clusters. This is very useful if you want to do some testing or reproduction of issues locally. Rather than setting it all up manually, _mlaunch_ will start the processes and connect the replica sets or shards together. Within a few seconds, you can have a complex environment running, like a 5 shard cluster, each shard consisting of a replica set with 2 nodes and an arbiter, authentication enabled, and any kinds of individual flags you want to pass onto the processes. _mlaunch_ also has options to start and stop individual instances or groups, and to view which ones are running in the current environment and which ones are down.



### How does it work?

Rather than going through all the features of each of the scripts, I'd just like to demonstrate two simple use cases. For a full list of features you can visit the [mtools wiki][wiki], which contains the manual and many usage examples. 

#### Use Case 1: Profiling your Queries with _mloginfo_

Let's say you find that some of your queries against MongoDB take really long, and possibly affect the performance of the database as a whole. To get an idea of where MongoDB spends most of its time, inspecting the "queries" section of _mloginfo_ is a good first step. Here is an example output showing just the "queries" section of _mloginfo_, created with the following command:

~~~
mloginfo mongod.log --queries
~~~

~~~
QUERIES

namespace                    pattern                                        count    min (ms)    max (ms)    mean (ms)    95%-ile (ms)    sum (ms)

serverside.scrum_master      {"datetime_used": {"$ne": 1}}                     20       15753       17083        16434         17039.3      328692
serverside.django_session    {"_id": 1}                                       562         101        1512          317           842.6      178168
serverside.user              {"_types": 1, "emails.email": 1}                 804         101        1262          201          684.85      162311
local.slaves                 {"_id": 1, "host": 1, "ns": 1}                   131         101        1048          310           819.5       40738
serverside.email_alerts      {"_types": 1, "email": 1, "pp_user_id": 1}        13         153       11639         2465          8865.2       32053
serverside.sign_up           {"_id": 1}                                        77         103         843          269           551.0       20761
serverside.user_credits      {"_id": 1}                                         6         204         900          369          763.75        2218
serverside.counters          {"_id": 1, "_types": 1}                            8         121         500          263           470.6        2111
serverside.auth_sessions     {"session_key": 1}                                 7         111         684          277           645.6        1940
serverside.credit_card       {"_id": 1}                                         5         145         764          368           705.0        1840
serverside.email_alerts      {"_types": 1, "request_code": 1}                   6         143         459          277           415.0        1663
~~~

Each line shows (from left to right) the namespace, the query pattern, and various statistics of this particular namespace/pattern combination. The rows are sorted by the "sum" column, descending. Sorting by sum is a good way to see where the database spent most of its time. In this example, we see that around half the total time is spent on a `$ne`-type queries on `serverside.scrum_master`, which are known to be inefficent as their excluding nature cannot benefit from an index and many documents have to be scanned. In fact, all of the queries took at least 15 seconds ("min" column). The "count" column also shows that only 20 of the queries were issued, yet these queries contributed to a large amount of the total time spent, more than double the 804 email queries on `serverside.user`. 

When optimizing queries and indexes, starting from the top of this list is a good idea as these optimizations will result in the highest gains in terms of performance.

#### Use Case 2: Visualizing Log Files with _mplotqueries_

Another way of looking at the performance of queries and other operations is to visualize them graphically. _mplotqueries_' scatter plot (the default) shows the duration of any operation (y-axis) over time (x-axis) and makes it easy to spot long-running operations. The following plot is generated with 

~~~
mplotqueries mongod.log
~~~
and then pressing `L` for "logarithmic" y-axis view:

![mplotqueries example plot: default]({{ site.url }}/assets/mtools-intro/mplotqueries_example1.png)

While most of the operations are sub-second (below the $10^3$ ms mark), the blue dots immediately stand out, reaching up to the hundreds and thousands of seconds. Clicking on one of the blue dots prints out the relevant log line to stdout:

~~~
Sat Apr  5 22:29:54 [conn99] command serverside.$cmd command: { getlasterror: 1, w: "majority" } ntoreturn:1 keyUpdates:0 reslen:112 1006370ms
~~~

The [`getlasterror`][getlasterror] command is used for write concern. In this case, it blocked until the write was replicated to a majority of nodes in the replica set, which took 16 minutes. That is of course an issue, and because this is a command and not a query (or the query part of an update), it didn't show up in the previous use case with `mloginfo --queries`. 

To investigate this further, we can overlay the current plot with an "rsstate" plot, that shows replica set status changes over time. The following two commands create an overlay of the two plots:

~~~
grep "majority" mongod.log | mplotqueries --overlay
mplotqueries mongod.log --type rsstate
~~~

![mplotqueries example plot: overlay]({{ site.url }}/assets/mtools-intro/mplotqueries_example2.png)

This shows that for each of the blocking "majority" `getlasterror`s, there seem to be some issues with replica set members being unavailable. The red vertical lines represent a node being `DOWN`, preceeding the yellow lines for a node being in `SECONDARY` state again, at which point the `getlasterror` commands finally succeed.

From here, the next step would be to look at all the log files of the replica set at that particular time and investigate why the secondaries became unavailable:

~~~
mlogfilter mongod.log mongod-sec.log mongod-arb.log --from Apr 5 20:15 --to +5min
~~~

This last command merges the log files of the three replica set members by time, each line prefixed with the filename, slices out a 5-minute window at the first instance of the issue and prints the lines back to stdout. 

We'll end this example here, but I hope this demonstrates the idea how _mtools_ can be used interactively for a root cause analysis. You look at the available data in various different ways, form a hypothesis, filter out noise (i.e. irrelevant log lines) and dig deeper.


### Where Can I Learn More?

These were just two simple examples of how _mtools_ can be used to diagnose MongoDB issues from log files. All scripts contain many more features and the best way to learn about them is to [download and install][install] _mtools_ and follow some of the examples on the [mtools wiki][wiki] page. _mtools_ is open source and available for download on [github][github]. It is also in the PyPi package index and can be installed via `pip`. Finally, for any questions, bug reports or feature requests, simply go to the [mtools github issues][issues] page and open a new issue. 


[d3]: http://d3js.org
[mongodb]: http://www.mongodb.com
[install]: https://github.com/rueckstiess/mtools/blob/master/INSTALL.md
[github]: https://github.com/rueckstiess/mtools
[wiki]: https://github.com/rueckstiess/mtools/wiki
[issues]: https://github.com/rueckstiess/mtools/issues
[getlasterror]: http://docs.mongodb.org/manual/reference/command/getLastError/



