---
layout: post
title: "Contributions to mtools"
date: 2013-09-03 00:12:02
categories: mtools contribute
---

[mtools][mtools] is a project I've been working on for some time now. It is a collection of command line scripts to parse, filter and visualize [MongoDB](http://www.mongodb.org/) log files, as well as spin up MongoDB test environments for issue reproduction and local testing. It's an open source project, available on [github][mtools] and I invite everyone to submit [feature requests, bug reports][mtools-issues], and [code contributions](https://github.com/rueckstiess/mtools/blob/master/tutorials/contributing.md).

In this blog entry, I want to demonstrate how anyone with some Python knowledge can contribute to the ongoing development of *mtools*. I'll go through all the required steps including getting a development version of *mtools* set up on your system, creating a git bugfix/feature branch and walking through an actual code change. 
Finally, we'll go through the process of pushing the change to a github fork and issuing a pull request.

The article also gives me a chance to highlight a change to the MongoDB log file format planned for MongoDB 2.6. If you have a MongoDB log parsing tool, you may need to make similar changes to the ones I describe below.

### Goal of this Code Change

MongoDB 2.6 introduces an option to change the timestamp format in log files; the ticket that tracks this change is&nbsp;[SERVER-7965](https://jira.mongodb.org/browse/SERVER-7965). In addition to the current (2.4) default called *ctime*, it will also support a local and a UTC version of the [ISO-8601](https://en.wikipedia.org/wiki/ISO_8601) standard. The log line parsing component in *mtools* needs to understand these additional formats, distinguish between them, and handle all three versions (with the old 2.2 version without milliseconds actually four versions) of the timestamp correctly.

The four timestamp formats that mtools needs to understand are therefore:

ctime-pre2.4
: Prior to v2.4 MongoDB's date format did not contain a year or milliseconds, example: `Wed Dec 31 19:00:00`

ctime
: Since v2.4 the ctime format contains milliseconds, example: `Wed Dec 31 19:00:00.000`

iso8601-utc
: This format uses the ISO-8601 standard and logs in UTC timezone, example:  `1970-01-01T00:00:00.000Z`

iso8601-local
: This format uses the ISO-8601 standard and logs in the machine's local timezone, example: `1969-12-31T19:00:00.000+0500`

### Getting a Development Version of mtools</span>

To start development with *mtools*, you need a development version. First, [fork the mtools repository](https://github.com/rueckstiess/mtools/fork) into your own github account (we'll use the placeholder `<username>` for this example, please replace it with your own github username). Then clone a copy to your local machine:

    git clone https://github.com/<username>/mtools
    
To set you up with a working development version of mtools, remove all previous installations (if you installed mtools with `pip`, you can use `sudo pip uninstall mtools`). Then change into the newly checked out mtools directory and run

    sudo python setup.py develop
    
This will install *mtools* in development mode, create all the hooks and scripts, but use that current directory as the only place for the source. That way, any change in the code will immediately apply and you can test your changes. If you get any errors when running above command, you may not have `setuptools` installed. If that is the case, go over to the [setuptools page](https://pypi.python.org/pypi/setuptools/1.1.1#installation-instructions) to get the latest version and try again.

Now you need to add the original "upstream" repository to pull in the latest changes:

    git remote add upstream https://github.com/rueckstiess/mtools
    git fetch upstream
    
You have two remote branches now: `origin` (pointing to your fork of mtools on github) and `upstream` (pointing to the original repository on github). To get a local develop branch as well, you need to check out and track your remote `develop` branch:

    git checkout -b develop origin/develop


### Setting Up a Feature/Bugfix Branch {#bugfix-branch}

All the steps up until now only have to be executed once. The next set of instructions applies to each individual code change / pull request that you want to do in the future.

If you want to work on a bug or feature implementation, always pull in the latest changes from upstream first:

    git checkout develop
    git pull upstream develop
    
This merges all the recent changes from upstream (the original repository) directly into your local copy. Now create a feature/bugfix branch that forks off the local develop branch:

    git checkout -b feature-76-timestamp develop
    
It's good practice to prefix the branch name with "feature" or "bugfix", include the issue number, and maybe a short descriptive word. The issue number here is the one from the corresponding [github issue](https://github.com/rueckstiess/mtools/issues). Now we can start work on this feature branch.

### Implementing the Change

Inside the `mtools` folder, you will find another nested `mtools` folder. That's just the way Python modules are usually set up. The inner `mtools` folder contains the actual *mtools* module. You will see a number of sub-folders, one for each command line tool (like `mlogfilter`, `mlaunch`, ...) and a few additional ones. This change, that parses log lines, affects all the tools, and the code for it is stored in `mtools/util/logline.py`. The `LogLine` class is implemented to evaluate all the elements in lazy fashion. This is because parsing a large number of log lines takes time, and if some or most of the fields are not needed for a specific application, we don't want to waste time *regex*-ing all the values out if they're not needed. The other perhaps unusual thing here is that all the methods are actually properties. This means that rather than calling something like `logline.get_datetime()`, you can just access `logline.datetime` as if it was a member variable. However, internally, the `datetime()` method is called, which lazily evaluates the date and time and stores it (by convention) in the actual member variable `_datetime`.

The datetime is extracted in two methods: `datetime()` and `_match_datetime_pattern()`. There are some assumptions that may not hold for the new timestamp format. First, it assumes that there are always 4 tokens (the strings you get when you split the log line on whitespace) that comprise the date and time. That only applies to the original ctime format and we need to change that. We also assume that these 4 tokens must appear within the first 10 tokens. Why not just look at the first 4? There are two reasons for that: [mlogmerge](https://github.com/rueckstiess/mtools#mlogmerge) might have placed a marker in front of the log line. And secondly, some MongoDB users use logging mechanisms (like [syslog](http://en.wikipedia.org/wiki/Syslog)), that prefix each log line with their own information, like a separate timestamp or a facility. *mtools* needs to be able to skip these additional tokens, but luckily the new timestamp formats won't have an effect on this. `datetime()` then hands off the actual parsing to a helper method called `_match_datetime_pattern()`. It looks like `datetime()` will work as is for the most part, and we can focus on changing `_match_datetime_pattern()` for the implementation.

This is the old implementation:

```python
def _match_datetime_pattern(self, tokens):
    """ Helper method that takes a list of tokens and tries to match the 
        datetime pattern at the beginning of the token list, i.e. the first
        few tokens need to match [weekday], [month], [day], HH:MM:SS  
        (potentially with milliseconds for mongodb version 2.4+). """
    
    if len(tokens) < 4:
        return None

    # return None
    weekday, month, day, time = tokens[:4]

    # check if it is a valid datetime
    if (weekday not in self.weekdays) or \
       (month not in self.months) or not day.isdigit():
        return None

    time_match = re.match(r'(\d{2}):(\d{2}):(\d{2})(\.\d{3})?', time)
    if not time_match:
        return None

    month = self.months.index(month)+1

    # extract hours, min, sec, millisec from time string
    h, m, s, ms = time_match.groups()

    # old format (pre 2.4) has no ms set to 0
    if ms == None:
        ms = 0
    else:
        ms = int(ms[1:])

    # convert to microsec for datetime
    ms *= 1000

    # assume this year. TODO: special case if logfile is not from current year
    year = datetime.now().year

    dt = datetime(int(year), int(month), int(day), int(h), int(m), int(s), ms)

    return dt
```

I won't go through every detail of the new implementation, but basically, the idea is: Find out with a number of quick and inexpensive format checks which of the four versions is the one at hand. Then parse the format appropriately. You may think that detecting the version over and over again for each log file is unnecessary overhead, but currently, there is no real concept of a log file, and each LogLine stands for itself. I also don't want to make any assumptions about consistency across a log file. If the user upgrades from 2.2 to 2.4, or if they decide to run with a different timestamp, the format would change within a file.

When searching for ways to interpret ISO-8601 timestamps in Python, I found that the [`dateutil`](http://labix.org/python-dateutil) module has a handy parser for this. The `dateutil` module has been in the Python standard library since 2.3, so we can use it without having to worry about incompatibilities. During tests with the `dateutil` parser in the Python shell, I discovered that it actually understands the ctime format as well. That makes it the perfect candidate for our code change.

I've also decided that it might be useful to store the detected format in the `LogLine` object. Other parts of *mtools* (for example the automatic version detection) might find this information useful. After committing this change, the datetime format is going to be available with the property `logline.datetime_format`, with a value of one of the four strings: `"ctime-pre2.4"`, `"ctime"`, `"iso8601-utc"` or `"iso8601-local"`.

The new `_match_datetime_pattern()` method (the main change) now looks like this:

```python
def _match_datetime_pattern(self, tokens):
    """ Helper method that takes a list of tokens and tries to match 
        the datetime pattern at the beginning of the token list. 

        There are several formats that this method needs to understand
        and distinguish between (see MongoDB's SERVER-7965):

        ctime-pre2.4:   Wed Dec 31 19:00:00
        ctime:          Wed Dec 31 19:00:00.000
        iso8601-utc:    1970-01-01T00:00:00.000Z
        iso8601-local:  1969-12-31T19:00:00.000+0500
    """
    # first check: less than 4 tokens can't be ctime
    assume_iso8601_format = len(tokens) < 4

    # check for ctime-pre-2.4 or ctime format
    if not assume_iso8601_format:
        weekday, month, day, time = tokens[:4]
        if len(tokens) < 4 or (weekday not in self.weekdays) or \
           (month not in self.months) or not day.isdigit():
            assume_iso8601_format = True

    if assume_iso8601_format:
        # sanity check, because the dateutil parser could interpret 
        # any numbers as a valid date
        if not re.match(r'\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}.\d{3}', \
                        tokens[0]):
            return None

        # convinced that this is a ISO-8601 format, the dateutil parser 
        # will do the rest
        dt = dateutil.parser.parse(tokens[0])
        self._datetime_format = "iso8601-utc" \
            if tokens[0].endswith('Z') else "iso8601-local"

    else:
        # assume current year (no other info available)
        year = datetime.now().year
        dt = dateutil.parser.parse(' '.join(tokens[:4]), \
                                   default=datetime(year, 1, 1))
        self._datetime_format = "ctime" \
            if '.' in tokens[3] else "ctime-pre2.4"

    return dt
```

All changes of the `logline.py` file can be seen in [this commit diff](https://github.com/rueckstiess/mtools/commit/098e358f0787354f274ec1e980fb9cc0a76e5258). As you can see, I had to change some other little details like the offset handling. Once the datetime is found in a log line, it aligns all other pieces of information, like the thread name, the type of operation (if present) etc. With both 4-token and 1-token datetimes, there were some additional changes necessary.

### Test Suite

Normally, at this point, we would write some tests for this new feature, to make sure that future code changes won't break the parsing again. We'd further run the test suite to make sure this particular change didn't break existing code. Unfortunately, the testing framework for mtools is not completed yet. This will be a major part of the 1.1 release, but writing thorough tests takes a lot of time. Once the test framework is merged back into the main develop branch and ready to use, I'll update this blog post to give some instructions where the tests go and how the suite is used.

### Committing the Change and Issuing a Pull Request

Back to the shell, we now have to commit the code change to the local feature branch. Check which files were changed and add them with `git add`:    

    git status
    git add mtools/util/logline.py
    
I would not recommend to use `git commit -a` (commit every change) all the time. It's easy to get used to the convenience of skipping the "add" step, but I prefer that extra check to make sure I only commit changes that I'm aware of and that belong to this particular feature. Now commit the change with a meaningful log message, and refer to the issue number (github will automatically link the commit to the issue).   

    git commit -m "Implemented logline parsing for new ISO-8601 timestamp formats. Fixes #76."
    
The change is now in your local feature branch. But it has to go to github for the Pull Request. Push the branch to your remote branch "origin".

    git push origin feature-76-timestamp
    
From the github web interface, you can now switch to this branch and issue a pull request against the upstream **develop** branch at `rueckstiess/mtools`. Please do not send pull requests to the master branch, it only contains release versions.

Once the pull request has been accepted, it will be merged in the develop branch, and you can pull the change from upstream develop and then delete your local and github feature/bugfix branch.

    git checkout develop
    git pull upstream develop
    git push origin --delete feature-76-timestamp
    git branch -d feature-76-timestamp
    
For other features and bug fixes, repeat the same process from ["Setting Up a Feature/Bugfix Branch"](#bugfix-branch) above.

[mtools]: https://github.com/rueckstiess/mtools
[mtools-issues]: https://github.com/rueckstiess/mtools/issues?state=open
