.. Copyright (C) 2011 James Aylett

Range queries
=============

.. todo:: check valueranges.rst to see if anything else needs moving across

I'm only interested in the 1980s
--------------------------------

In the museums dataset we used in our earlier examples, there is a
field `DATE_MADE` that tells us when the object in question was made,
so one of the natural things people might want to do is to only search
for objects made in a particular time period. Suppose we want to
extend our original system to allow that, we're going to have to do a
number of things.

 1. Parse the field from the data set to turn it into something consistent;
    at the moment it includes years, year ranges ("1671-1700"), approximate
    years ("c. 1936") and commentary ("patented 1885", or "1642-1649
    (original); 1883 (model)"). Additionally, some records have no
    information about when the object was made.
 2. Store that information in the Xapian database.
 3. Provide a way during search of specifying a date range to constrain to.

If we look through the other fields in the data set, there are more
that could be useful for range queries: we could extract the longest
dimension from `MEASUREMENTS` to enable people to restrict to only
very large or very small objects, for instance.

We'll see how to perform range searches across both those dimensions,
and then we'll look at how to cope with full dates rather than just
years.


How Xapian supports range queries
---------------------------------

If you think back to when we introduced the query concepts behind
Xapian, you'll remember that one group of query operators is
responsible for handling *value ranges*: `VALUE LE`, `VALUE GE`
and `VALUE RANGE`. So we'll be tackling range queries by using
document values, and constructing queries using these operators to
restrict matches suitably.

Since we want to expose this functionality generally to users, we want
them to be able to type in a query that will include one or more range
restrictions; the QueryParser contains support for doing this, using
*value range processors*, subclasses of `ValueRangeProcessor`. Xapian
comes with some standard ones itself, or you can write your own.

Since document values are stored as strings in Xapian, and the
operators provided perform string comparisons, we need a way of
converting numbers to strings to store them. For this, Xapian provides
a pair of utility functions: `sortable_serialise` and
`sortable_unserialise`, which convert between floating point numbers
(strictly, each works with a `double`) and a string that will sort in
the same way and so can be compared easily.

You can fill `#x_value_name.type` with the `float` value and pass this record
to `xapian_server:open/2` as a parameter.

.. code-block:: erlang
    #x_value_name{slot = 1, name = admitted_year, type = float}.

After this, serializition will be done automaticly.

Creating the document values
----------------------------

We need a new version of our indexer. This one is
`bin/index_ranges.escript`, and creates document values from both
`MEASUREMENTS` and `DATE_MADE`. We'll put the largest dimension in
value slot 0 (fortunately the data is stored in millimetres and
kilograms, so we can cheat a little and assume that dimensions will
always be larger than weights), and a year taken from `DATE_MADE` into
value slot 1 (we choose the first year we can parse, since it can
contain such a variety of date formats).

This command reads the data file and fills the database::

    $ ./bin/index_ranges.escript ./priv/100-objects-v1-utf8.csv ./priv/test_db/ranges

We can check this has created document values using `delve`::

    $ delve -V0 ./priv/test_db/ranges
    Value 0 for each document: 5:?? 10:?F 11:?? 12:?P 15:? 19:?t 20:?? 21:? 24:?: 25:?? 26:?? 27:?X 29:?D 30: 31:?@ 33:?` 34:?0 35:?? 36:? 37:?? 38:?( 39:?T 42:?2 45:?@ 46:?P 50:?? 51:?P 52:̡ 54:è 55:?? 56:?P 59:?` 61:?( 62:?@ 64:?? 66:?? 67:?` 68:?D33333@ 69:? 70:?? 71:˨ 72:? 73:??fffff? 74:??fffff? 75:?$?????? 76:¿33333@ 77:?>33333@ 78:?? 79:? 80:?P 81:?@ 84:?? 86:?~ 87:?? 88:?(?????? 89:??33333@ 90:??33333@ 91:?| 93:?( 94:?` 97:?? 98:?h 100:? 101:?V 102:??

All the '?' characters are because `delve` doesn't know to run
`sortable_unserialise` to turn the strings back into numbers.

Searching with ranges
---------------------

All we need to do once we've got the document values in place is to
tell the QueryParser about them. The simplest value range processor is
`StringValueRangeProcessor`, but here we need two
`NumberValueRangeProcessor` instances.

To distinguish between the two different ranges, we'll require that
dimensions must be specified with the suffix 'mm', but years are just
numbers. For this to work, we have to tell QueryParser about the value
range with a suffix first:

.. code-block:: erlang
    Procs = [xapian_resource:number_value_range_processor(0, "mm", suffix)
            ,xapian_resource:number_value_range_processor(1) ],
    Query = #x_query_string{
        value = QueryStr,
        parser = #x_query_parser{value_range_processors = Procs}}.

We use a `xapian_resource:number_value_range_processor/X` function for creating
a resource constructor of the `NumberValueRangeProcessor` object.

The first call has a final parameter of `suffix` (or `false`) to say that
'mm' is a suffix (the default is for it to be a prefix). When using the empty
string, as in the second call, it doesn't matter whether you say it's
a suffix or prefix, so it's convenient to skip that parameter.


This is implemented in `bin/search_ranges.escript`, which also
modifies the output to show the measurements and date made fields as
well as the title.

We can now restrict across dimensions using queries like '..50mm'
(everything at most 50mm in its longest dimension), and across years
using '1980..1989'::

    $ ./bin/search_ranges.escript priv/test_db/ranges ..50mm
     1: #031   50.0   1588   |Portable universal equinoctial sundial, |
     2: #073  44.45   1701   |Universal pocket sundial                |
     3: #074  44.45   1596   |Sundial, made as a locket, gilt metal, p|

    $ ./bin/search_ranges.escript priv/test_db/ranges 1980..1989
     1: #050  105.0   1984   |Quartz Analogue "no battery" wristwatch |
     2: #051   85.0   1984   |Analogue quartz clock with voice control|

You can of course combine this with 'normal' search terms, such as all
clocks made from 1960 onwards::

    $ ./bin/search_ranges.escript priv/test_db/ranges "clock 1960.."
     1: #052 1185.0   1974   |Reconstruction of Dondi's Astronomical C|
     2: #051   85.0   1984   |Analogue quartz clock with voice control|
     3: #009  380.0   1973   |Copy  of a Dwerrihouse skeleton clock wi|

and even combining both ranges at once, such as all large objects from the 19th century::

    $ ./bin/search_ranges.escript priv/test_db/ranges "clock  1000..mm 1800..1899"
     1: #024 1850.0   1845   |Regulator Clock with Gravity Escapement |

Note the slightly awkward syntax *1000..mm*. The suffix must always go
on the end of the entire range; it may also go on the beginning (so
you can do *1000mm..mm*). Similarly, you can have *100mm..200mm* or
*100..200mm* but not *100mm..200*. These rules are reversed for
prefixes.

If you get the rules wrong, the QueryParser will raise a
`QueryParserError`, which in production code you could catch and
either signal to the user or perhaps try the query again without the
`ValueRangeProcessor` that tripped up::

    $ ./bin/search_ranges.escript priv/test_db/ranges 1000mm..
    escript: exception error: {x_error,<<"QueryParserError">>,
                              <<"Unknown range operation">>}
      in function  xapian_server:client_error_handler/1
        (src/xapian_server.erl, line 1110)
      in call from erl_eval:do_apply/6 (erl_eval.erl, line 572)
    ...


Handling dates
--------------

To restrict to a date range, we need to decide how to both store the
date in a document value, and how we want users to input the date
range in their query. `DateValueRangeProcessor`, which is part of
Xapian, works by storing the date as a string in the form 'YYYYMMDD',
and can take dates in either US style (month/day/year) or European
style (day/month/year).

To show how this works, we're going to need to use a different
dataset, because the museums data only gives years the objects were
made in; we've built one using data on the fifty US states, taken from
Wikipedia infoboxes on 5th November 2011 and then tidied up a small
amount. The CSV file is `data/states.csv`, and the code that did most
of the work is `code/python/from_wikipedia.py`, using a list of
Wikipedia page titles in `data/us_states_on_wikipedia`. The CSV is
licensed as Creative Commons Attribution-Share Alike 3.0, as per
Wikipedia.

We need a new indexer for this as well, which is
`bin/index_ranges2.escript`. It stores two numbers using
`sortable_serialise`: year of admission in value slot 1 and population
in slot 3. It also stores the date of admission as 'YYYYMMDD' in
slot 2. We'll look at just the date ones for now, and come back to the
others in a minute.

There isn't any new code in this indexer that's specific to Xapian,
although there's a fair amount of work to turn the data from Wikipedia
into the forms we need. We use the indexer in the same way as previous
ones::

    $ ./bin/index_ranges2.escript priv/states.csv priv/test_db/ranges2

With this done, we can change the set of value range processors we
give to the QueryParser.

Here `DateValueRangeProcessor` is created:
    
.. code-block:: erlang
    xapian_resource:date_value_range_processor(2, 1860, true).

The `DateValueRangeProcessor` is working on value slot 2, with an
"epoch" of 1860 (so two digit years will be considered as starting at
1860 and going forward as far 1959). The second parameter is whether
it should prefer US style dates or not; since we're looking at US
states, we've gone for US dates. 

This code can be rewritten using a name of the slot as a first parameter:

.. code-block:: erlang
    xapian_resource:date_value_range_processor(admitted, 1860, true).

The `NumberValueRangeProcessor` is as we saw before.

This enables us to search for any state that talks about the Spanish
in its description::

    ./bin/search_ranges2.escript priv/test_db/ranges2 spanish
    1: #004          State of Montana       18891108            989415
    2: #019            State of Texas       18451229          25145561

or for all states admitted in the 19th century::

    $ ./bin/search_ranges2.escript priv/test_db/ranges2 "1800..1899"
     1: #001       State of Washington      18891111           6744496
     2: #002         State of Arkansas      18360615           2915918
     3: #003           State of Oregon      18590214           3831074
     4: #004          State of Montana      18891108            989415
     5: #005                     Idaho      18900703           1567582
     6: #006           State of Nevada      18641031           2700551
     7: #007       State of California      18500909          37253956
     8: #009             State of Utah      18960104           2763885
     9: #010          State of Wyoming      18900710            563626
    10: #011         State of Colorado      18760801           5029196

That uses the `NumberValueRangeProcessor` on value slot 1, as in our
previous example. Let's be more specific and ask for only those
between November 8th 1889, when Montana became part of the Union, and
July 10th 1890, when Wyoming joined::

    $ ./bin/search_ranges2.escript priv/test_db/ranges2 "11/08/1889..07/10/1890"
     1: #001       State of Washington      18891111           6744496
     2: #004          State of Montana      18891108            989415
     3: #005                     Idaho      18900703           1567582
     4: #010          State of Wyoming      18900710            563626

That uses the `DateValueRangeProcessor` on value slot 2; it can't cope
with year ranges, which is why we indexed to both slots 1 and 2.

