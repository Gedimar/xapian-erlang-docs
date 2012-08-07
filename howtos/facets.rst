.. Copyright (C) 2007,2010,2011 Olly Betts
.. Copyright (C) 2009 Lemur Consulting Ltd
.. Copyright (C) 2011 Richard Boulton
.. Copyright (C) 2011 Justin Finkelstein
.. Copyright (C) 2011 James Aylett

======
Facets
======

Xapian provides functionality which allows you to dynamically generate
complete lists of values which feature in matching documents. For example,
colour, manufacturer, size values are good candidates for faceting.

There are numerous potential uses this can be put to, but a common one is
to offer the user the ability to narrow down their search by filtering it
to only include documents with a particular value of a particular category.
This is often referred to as `faceted search`.


Implementation
==============
Faceting works against information stored in document value slots [link to
value slots] and, when executed, provides a list of the unique values for
that slot together with a count of the number of times each value occurs.


Indexing
--------

No additional work is needed to implement faceted searching, except to
ensure that the values you wish to use in facets are stored as
document values.

We use two value slots: 0 contains the collection, and 1
contains the name of whoever made the object. We know from the
documentation of the dataset that both from fixed and curated lists,
so we don't have to worry about normalising the values before using
them as facets. Let's run that to build a dataset with document values
suitable for faceting::

    $ ./bin/index_filters.escript ./priv/100-objects-v1-utf8.csv ./priv/test_db/index_filters


Querying
--------

.. todo:: explain what goes on here, ie why we use a MatchSpy

Faceting information is collected, using `Xapian::ValueCountMatchSpy` object,
that can be created, calling

.. code-block:: erlang

    MatchSpyRes = xapian_match_spy:value_count(Server, SlotNumberOrName).

This function returns a resource, that collects information during
a `MatchSet` generation. 

.. code-block:: erlang
    MSetParams = #x_match_set{
    enquire = EnquireRes,
    spies = [MatchSpyRes]},
    MSetRes = xapian_server:match_set(Server, MSetParams).

A `MatchSpy` resource can be used with `xapian_server:match_set/2` only once.
Then, you can extract statistics using few functions, defined inside 
the `xapian_term_qlc` module.
    
.. code-block:: erlang

    value_count_match_spy_table(Server, SpyRes, Meta).
    top_value_count_match_spy_table(Server, SpyRes, Limit, Meta).

The first function will give you facets in alphabetical order, not in
order of frequency. if you want to show the most frequent first you
should use the second function.

We use QLC for retrieving collected information. Each unique value is described
by a record. The record can have only two fields: `value` and `freq`.
`Meta` describes the format of the record. It can be created, using:

.. code-block:: erlang

    xapian_term_record:record(RecName, RecFields).

For example:

.. code-block:: erlang

    %% On the top of the file
    -record(spy_term, {value :: unicode:unicode_binary(), 
                        freq :: non_neg_integer()}).

    %% Inside the function
    Meta = xapian_term_record:record(spy_term, 
            record_info(fields, spy_term)).


In the example, we're faceting on value slot 1, which is the object maker. 
After you get the MSet, you can ask the spy for the facets it found,
including the frequency. Note that although we're generally only
showing ten matches, we use a field of `#x_match_spy{}` called
`check_at_least`, so that the entire dataset is considered and the facet
frequencies are correct. See `Limitations`_ for some discussion of the
implications of this. Here's the output::

    ./bin/search_facets.escript ./priv/test_db/facets clock
    1: docid=44 Two-dial clock by the Self-Winding Clock Co; as used on the
    2: docid=96 Clock with Hipp pendulum (an electric driven clock with Hipp
    3: docid=12 Assembled and unassembled EXA electric clock kit
    4: docid=98 'Pond' electric clock movement (no dial)
    5: docid=83 Harrison's eight-day wooden clock movement, 1715.
    6: docid=5 "Ever Ready" ceiling clock
    7: docid=39 Electric clock of the Bain type
    8: docid=61 Van der Plancke master clock
    9: docid=64 Morse electrical clock, dial mechanism
    10: docid=52 Reconstruction of Dondi's Astronomical Clock, 1974
    Facet: Bain, Alexander; count: 2
    Facet: Bloxam, J. M.; count: 1
    Facet: Braun (maker); count: 1
    Facet: British Horo-Electric Ltd. (maker); count: 1
    Facet: EXA; count: 1
    Facet: Ever Ready Co. (maker); count: 2
    Facet: Ferranti Ltd.; count: 1
    Facet: Harrison, John (maker); count: 1
    Facet: Hipp, M.; count: 1
    Facet: La Prision Cie; count: 1
    Facet: Lund, J.; count: 1
    Facet: Morse, J. S.; count: 1
    Facet: Self Winding Clock Company; count: 1
    Facet: Self-Winding Clock Co. (maker); count: 1
    Facet: Synchronome Co. Ltd. (maker); count: 2
    Facet: Thwaites and Reed Ltd.; count: 1
    Facet: Thwaites and Reed Ltd. (maker); count: 1
    Facet: Viviani, Vincenzo; count: 1
    Facet: Whitefriars Glass Ltd. (maker); count: 1

If you want to work with multiple facets, you can register multiple
passing a list of spies. Although each additional one will have some 
performance impact.

.. code-block:: erlang

    MSetParams = #x_match_set{
        enquire = EnquireRes,
        spies = [MatchSpyRes1, MatchSpyRes2]}.

Finally, the resource should be deallocated.

.. code-block:: erlang

    xapian_server:release_resource(MatchSpyRes).


Restricting by Facets
---------------------

If you're using the facets to offer the user choices for narrowing down
their search results, you then need to be able to apply a suitable filter.

For a single value, you could use `#x_query_value{slot = Slot, value = Value}`, 
or `Xapian::MatchDecider`, but it's probably most
efficient to also index the categories as suitably prefixed boolean terms
and use those for filtering.


Limitations
===========

The accuracy of Xapian's faceting capability is determined by the number
of records that are examined by Xapian whilst it is searching. You can
control this number by specifying the `#x_match_set.checkatleast` value;
however it is important to be aware that increasing this number may have an
effect on overall query performance.


In Development
==============
Some additional features currently in development may benefit users of
facets. These are:

    * Multiple values in slots: this will allow you to have a single value slot
      (e.g. colour) which contains multiple values (e.g. red, blue).  This will
      also allow you to create a facet by colour which is aware of these
      multiple values, giving counts for both red and blue.

    * Bucketing: this provides a means to group together numeric facets, so that
      a single facet can contain a range of values (e.g. price ranges).
