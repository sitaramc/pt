pt - simple, distributed (git-aware) project tracker (CLI)

I have a somewhat distributed team, though not too large.  We don't have a lot
to keep track of, so ethercalc has been working well enough for some time.
But now we feel the need for a bit more power (searching, tags, ...), and
especially offline access (which eliminates most of the web-based tools).

Basically, it needs to be distributed, using git as both storage *and*
transport.  Everything else is optional, even a GUI.

My team uses [gitolite][] anyway, so using git as the backend for the tracker
would make access control trivial.  That's a huge bonus for me.  (You don't
have to use gitolite.  Any git server -- gitlab, gogs, github, whatever --
will give you the same benefit, because they all do access control.)

----

Here's what I came up with.  It's loosely inspired by [gi][], which I did try.
It was pretty good, but was too closely tied to git -- git commit hashes are
used as issue IDs.  That might not sound like a problem right now, but I think
things would break if I ever wanted/needed to do a rebase.

So I rolled my own.  For the ID, I used the `uuidgen -t` command (I don't have
to worry about Windows and Mac, and this command seems to be present on all
the Linuxes my team uses).  I looked at the specifications of uuidgen, and
figured out how much of it I could cut off without risking duplicates,
prefixed a `YYYY-MM-DD-` and that's my ID.

[gitolite]: https://github.com/sitaramc/gitolite
[gi]: https://github.com/dspinellis/gi

# names, things, and names of things

Each entry has an ID, which looks like `2018-03-31-0d8ef858-34a4`.  Any
command that expects an ID can use an initial abbreviation of the part of the
ID after the date; for example the above ID can be referred to as `0d` or
maybe `0d8e` if you have a lot of entries.  This is pretty much like git,
except the full ID starts with a date before going all random on you :)

Each entry has a 1-line "title", 0-or-more lines of "details", and an optional
series of 1-line events in a "log".

Each entry also has 0 or more "tags".  The tag "closed" is special, otherwise
anything goes (characters allowed are alphanumerics and `_@.:-`).

Finally, each entry may have one or more attachments.

# how to use

Just put this program in your `$PATH` and go.  A sample session follows.

At the bottom of this README is a copy of the inline help text embedded within
the script, which you get by running `pt help` or `pt -h`.

# sample session

## clone

First, have your admin create an empty repo on the git server and give
permissions to you and your team.

Clone this [empty] repo to `$HOME/.project-tracker`.

    git clone myserver:my-pt.git .project-tracker

`$HOME/.project-tracker` is the default location that the `pt` command
assumes.  You can override it by setting the `PROJECT_TRACKER_DIR` environment
variable.  You could even have multiple project trackers, by using a wrapper
that sets the environment variable to the right value.

## add items/issues

Next, start adding data.  (Note: no need to `cd` to `~/.project-tracker` etc.)

    $ pt new issue aa

When you do this, an editor pops up, containing some text ("# by: $USER").
You can append to it, or replace it completely, as you wish, then save and
exit.  You will see something like this:

    created 2018-03-31-0d8ef858-34a4

Similarly, you can add more issues.  After a few issues or items are added,
try listing them:

    $ pt
    2018-03-31-0d8ef858-34a4 |                        | issue aa
    2018-03-31-6da36b5c-34a4 |                        | issue bb
    2018-03-31-71ed48a4-34a4 |                        | item cc
    2018-03-31-771b484e-34a4 |                        | item dd

## list by matching in title

**NOTE**: the `list` command can be abbreviated as `l` for convenience.

You can search on the title using a regular expression:

    $ pt list /issue
    2018-03-31-0d8ef858-34a4 |                        | issue aa
    2018-03-31-6da36b5c-34a4 |                        | issue bb

You can have multiple regexes, and some of them can be negated also:

    $ pt list /issue -/aa
    2018-03-31-6da36b5c-34a4 |                        | issue bb

You can also search on the combined content of the title, detail, and log (but
we'll see this later).

## show an issue in full

**NOTE**: the `show` command can be abbreviated as `s` for convenience.

You can show all the details of an issue, including tags, details, any
attached files, and log events.

You can do this by using an item's short ID, rather like git's hashes; simply
take the first few characters after the date and the hyphen:

    $ pt show 6d
    title:  issue bb
    tags:   
    id:     2018-03-31-6da36b5c-34a4

    # by: sitaram
    detail for issue 2

Or you can use its full ID:

    $ pt show 2018-03-31-71ed48a4-34a4
    title:  item cc
    tags:   
    id:     2018-03-31-71ed48a4-34a4

    # by: sitaram
    detail for issue 3

We haven't added any log events yet; if we had we'd see them too.

## add tags

You can add some tags:

    $ pt tag 0d foo
    $ pt tag 6d foo
    $ pt tag 6d bar
    $ pt tag 71 bar

then try listing all the items again:

    $ pt
    2018-03-31-0d8ef858-34a4 | foo                    | issue aa
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb
    2018-03-31-71ed48a4-34a4 | bar                    | item cc
    2018-03-31-771b484e-34a4 |                        | item dd

## list by matching tags

You can list by tags, same as regexes, including negation.  Notice all the
combinations we're trying, and compare them to the full list above:

    $ pt list foo
    2018-03-31-0d8ef858-34a4 | foo                    | issue aa
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb

    $ pt list bar
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb
    2018-03-31-71ed48a4-34a4 | bar                    | item cc

    $ pt list foo bar
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb

    $ pt list foo -bar
    2018-03-31-0d8ef858-34a4 | foo                    | issue aa

    $ pt list bar -foo
    2018-03-31-71ed48a4-34a4 | bar                    | item cc

    $ pt list -bar -foo
    2018-03-31-771b484e-34a4 |                        | item dd

## combining regex match and tags

You can combine both kinds of searches shown before:

    $ pt list /issue foo
    2018-03-31-0d8ef858-34a4 | foo                    | issue aa
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb

    $ pt list /issue bar
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb

    $ pt list /issue -bar
    2018-03-31-0d8ef858-34a4 | foo                    | issue aa

In fact, **you can specify any number of regexes and any numeber of tags**;
they will all be AND-ed together.

    $ pt /issue -/aa foo
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb

    $ pt /issue -/aa foo -bar
    <no output>

## list all tags

You can list all the tags the system knows, just in case you forgot (the
spelling of) one.

    $ pt tags
    bar
    foo

## log an event

You can log an event:

    $ pt log 6d alice sent an email about this to bob

then show the full item:

    $ pt show 6d
    title:  issue bb
    tags:   bar, foo
    id:     2018-03-31-6da36b5c-34a4

    # by: sitaram
    detail for issue 2

    --------------------------------------------------------------------------------
    LOG:
    2018-03-31      (sitaram) tag foo added
    2018-03-31      (sitaram) tag bar added
    2018-03-31      (sitaram) alice sent an email about this to bob

Oh hey, I forgot to mention that adding a tag is recorded as an event as well,
so there you are!  (**Reminder**: `sitaram` is the userid I am running this in; we
simply record `$USER` just for convenience.)

Let's add an event to the other issue:

    $ pt log 0d logging something here

## check the history

**NOTE**: the `history` command can be abbreviated as `h` for convenience.

You can check the history of one or more items, using either an

*   item's ID (full or abbreviated), or

*   a search expression that is similar to the list command's search syntax
    shown above, **except** that it **must** start with a regex (i.e., it
    cannot start with a tag):

Here's one with just a regex:

    $ pt history /issue
    2018-03-31-0d8ef858-34a4 | foo                    | issue aa
    2018-03-31      (sitaram) tag foo added
    2018-03-31      (sitaram) logging something here

    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb
    2018-03-31      (sitaram) tag foo added
    2018-03-31      (sitaram) tag bar added
    2018-03-31      (sitaram) alice sent an email about this to bob

Notice this showed us two items.  For each item, we see an ID, its tags, the
title, and then on succeeding lines the log events.

We logged all these events on the same day, from the same user, otherwise
you'd see different dates and different users (instead of just `sitaram`).

## handle attachments

Sometimes you need to attach a file to an issue (we'll choose issue ID `0d`):

    $ pt attach 0d something.png

List attachments:

    $ pt files 0d
    something.png

Get and open an attachment.  You don't have to supply the full name; any part
of it that makes it unique (among all the attachments for *that* ID) will do.

     $ pt open 0d thing
     you have 'something.png' in your current directory; will not overwrite, sorry!

Oops.  Oh well...

    $ cd /tmp
    $ pt open 0d thing

...and the file will be copied to your current directory, and `xdg-open` will
be run on it.  (You can specific some other program instead of `xdg-open` by
setting the `PROJECT_TRACKER_XDG_OPEN` environment variable.)

Oh and in case you forgot what IDs have attachments on them:

    $ pt
    2018-03-31-0d8ef858-34a4 | [1] foo                | issue aa
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb
    2018-03-31-71ed48a4-34a4 | bar                    | item cc
    2018-03-31-771b484e-34a4 |                        | item dd

Notice that `[1]` in the first entry?  That's the number of files this ID has.

If you remember part of the filename ("thing" in this case) but too many IDs
to go running `pt files` on each:

    $ pt //file.*thing.*added
    2018-03-31-0d8ef858-34a4 | [1] foo                | issue aa

This shows us the item ID.  Or you can use the history command:

    $ pt history //file.*thing.*added
    2018-03-31-0d8ef858-34a4 | foo                    | issue aa
    2018-03-31      (sitaram) tag foo added
    2018-03-31      (sitaram) logging something here
    2018-03-31      (sitaram) file /home/sitaram/something.png added

where the last log line shows you the name of the file you added (the actual
filename is of course only the basename).

Incidentally this is also the promised example of searching the title,
details, *and* log files.  Here we are taking advantage of the fact that the
file addition is recorded as an event in the log file.

Please see the "direct edit" section later for some cautions.

## close an issue/item

    $ pt
    2018-03-31-0d8ef858-34a4 | [1] foo                | issue aa
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb
    2018-03-31-71ed48a4-34a4 | bar                    | item cc
    2018-03-31-771b484e-34a4 |                        | item dd

    $ pt close 71

    $ pt
    2018-03-31-0d8ef858-34a4 | [1] foo                | issue aa
    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb
    2018-03-31-771b484e-34a4 |                        | item dd

    $ pt list closed
    2018-03-31-71ed48a4-34a4 | bar, closed            | item cc

## direct edit

You can directly edit any item.  The three files "title", "details" and "log"
will open up in your editor (or vim, if neither `$EDITOR` nor `$VISUAL` are
set).  Make any changes you want and save them.

Note that the log file is actually free format.  You can add whatever you want
to it, and the `show` command will show it, but the history command will only
show those lines that start with a date.

    $ pt edit 0d
    3 files to edit

I've just edited a log entry, which you can see after the second log line
below:

    $ pt show 0d
    title:  issue aa
    tags:   foo
    id:     2018-03-31-0d8ef858-34a4
    files:
            something.png

    # by: sitaram
    detail for issue 1

    --------------------------------------------------------------------------------
    LOG:
    2018-03-31      (sitaram) tag foo added
    2018-03-31      (sitaram) logging something here
        added an extra line of explanation for above log entry
    2018-03-31      (sitaram) file /home/sitaram/something.png added

But when you try the `history` command, you won't see that extra line:

    $ pt history /issue
    2018-03-31-0d8ef858-34a4 | foo                    | issue aa
    2018-03-31      (sitaram) tag foo added
    2018-03-31      (sitaram) logging something here
    2018-03-31      (sitaram) file /home/sitaram/something.png added

    2018-03-31-6da36b5c-34a4 | bar, foo               | issue bb
    2018-03-31      (sitaram) tag foo added
    2018-03-31      (sitaram) tag bar added
    2018-03-31      (sitaram) alice sent an email about this to bob

## `sync` and `git`

For convenience, `pt sync` will run the following commands:

    git add -A
    git commit -m $ENV{USER}
    git pull
    git push origin master

Also for convenience, you can run any `git` command on the project tracker
directory, for example `pt git log --stat`.  Naturally you don't need to run
the `sync` command if you don't wish to -- just run whatever git commands you
need to.

# inline help text

Here's the inline help, accessed by running `pt help` or `pt -h`:

    Simple git-aware project tracker.  Each item (issue, note, whatever you want
    to call it) has a 1-line "title" file, a free-form "details" file, and a
    mostly free-form "log" file.  It also has zero or more "tags".  The tag
    "closed" has a special meaning (which should be obvious).

    Usage:
        pt new <title>          # also opens an editor to take "details"
        pt tag <ID> <tag>       # add a tag
        pt tag <ID> -<tag>      # remove a tag (note the "-" sign)
        pt tags                 # list of all tags ever used in system (to check typos!)
        pt log <ID> <text>      # append text to the "log" file of <ID>

        pt                      # same as 'pt list' without arguments
        pt list                 # list all open items (i.e., same as 'pt list -closed')
        pt list <arguments>             # see LIST MODE below
        pt show <ID|search terms>       # see SHOW MODE below
        pt hist <ID|search terms>       # see HIST MODE below
            (the above 3 commands can be abbreviated to 'l', 's', and 'h', respectivey)

        pt attach <ID> <file>   # attach a file to item
        pt files <ID>           # list files attached to an item
        pt open <ID> <file>     # get file from item to current directory and open it
                                # (partial filenames are OK, but should be unique)

        pt close <ID>           # adds the "closed" tag
        pt edit <ID>            # opens editor on title, details, and log files
                                #   !! USE WITH CARE !!

        pt sync                 # add and commit local changes, pull, and push
        pt git <arguments>      # run any git command

    NOTE on <ID>: You need not supply the full ID; just the first few characters
    after the date should be fine (as much as is needed to establish uniqueness)

    LIST MODE

    -   List mode shows each item in a single line, formatted loosely as follows:

            ID  |  list of tags  |  title

        The list of tags field also has a number in brackets, like [2], showing
        the number of files attached to that item, if any.

    -   List mode with arguments has the following variants.  In all cases, unless
        the tag "closed" is supplied, only open items are listed.

        pt list /regex          # list items where regex matches in title
        pt list //regex         # same, but match in title, details, or log
        pt list tag1            # list items containing the tag

        In particular, note that list mode cannot take a single ID as an argument.
        It has to be some kind of search.

    -   You can use any number of regexes and tags; they are all ANDed together.

    -   You can also have negations; e.g., '-tag2' or '-/regex'.  As a final
        example, '/foo -//bar tag1 -tag2' lists items matching foo in the title,
        and *not* matching bar in the title, details, or log, and which contain
        tag1 and do not contain tag2.

    SHOW MODE

    -   Show mode shows each item in full, in a self explanatory format.

    -   Show mode can take a single ID as an argument and will then show only that
        item.  It can also take search arguments like LIST MODE above, except that
        you *must* have a regex as the first item; if you start the expression
        with a tag it won't work.

    HIST MODE

    *   History mode shows the first line in the same format as the list command,
        then shows all lines from the "log" file that start with a date
        (YYYY-MM-DD); in effect, this is an event log for the item.

    -   History mode search terms have the same rules as SHOW MODE above


