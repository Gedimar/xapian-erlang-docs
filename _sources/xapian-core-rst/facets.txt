
.. Copyright (C) 2007,2010,2011 Olly Betts
.. Copyright (C) 2009 Lemur Consulting Ltd
.. Copyright (C) 2011 Richard Boulton

=======================
Xapian Faceting Support
=======================

.. contents:: Table of contents

Introduction
============

Xapian provides functionality which allows you to dynamically generate complete
lists of category values which feature in matching documents.  There are
numerous potential uses this can be put to, but a common one is to offer the
user the ability to narrow down their search by filtering it to only include
documents with a particular value of a particular category.  This is often
referred to as ``faceted search``.

You may have many multiple facets (for example colour, manufacturer, product
type) so Xapian allows you to handle multiple facets at once.

How to use Faceting
===================

Indexing
--------

When indexing a document, you need to add each facet in a different numbered
value slot.  As described elsewhere in the documentation, each Xapian document
has a set of "value slots", each of which is addressed by a number, and can
contain a value which is an arbitrary string.

The ``#x_value{}`` record can be used to put values into a
particular slot.  So, if you had a database of books, you might put "price"
facet values in slot 0, say (serialised to strings using
the ``float`` value type, or some similar function), "author" facet
values in slot 1, "publisher" facet values in slot 2 and "publication type"
(eg, hardback, softback, etc) values in slot 3.

.. code-block:: erlang

    Params = [write, create, overwrite,
        #x_value_name{slot = 0, name = price, type = float},
        #x_value_name{slot = 1, name = author},
        #x_value_name{slot = 2, name = publisher},
        #x_value_name{slot = 3, name = publication_type},
    ],
    {ok, Server} = xapian_server:open(Path, Params),
    Document =
        [ #x_value{slot = price, value = 100}
        , #x_value{slot = author, value = "Joe Armstrong"}
        , #x_value{slot = publisher, value = "The Pragmatic Bookshelf"}
        , #x_value{slot = publication_type, value = "softback"}
        ],
    xapian_server:add_document(Server, Document).


Searching
---------

Finding Facets
~~~~~~~~~~~~~~

At search time, for each facet you want to consider, you need to get a count of
the number of times each facet value occurs in each slot; for the example
above, if you wanted to get facets for "price", "author" and "publication type"
you'd want to get the counts from slots 0, 1 and 3.

This can be done using a value count match spy. It can be created, calling 
``xapian_match_spy:value_count/2``.
Spies are used by setting the ``spies`` field of the ``#x_match_set{}`` record
to a list of resources for each value slot you want to get facet counts 
for, like so:

.. code-block:: erlang

    SpySlot0 = xapian_match_spy:value_count(Server, price),
    SpySlot1 = xapian_match_spy:value_count(Server, author),
    SpySlot3 = xapian_match_spy:value_count(Server, publication_type),

    Query = "",
    EnquireResourceId = xapian_server:enquire(Server, Query),
    MSetParams = #x_match_set{
        enquire = EnquireResourceId,
        from = 0,
        max_items = 10,
        check_at_least = 10000,
        spies = [SpySlot0, SpySlot1, SpySlot3]},
    MSetResourceId = xapian_server:match_set(Server, MSetParams).

The ``10000`` in the ``check_at_least`` field tells Xapian to check at least
10000 documents, so the MatchSpy objects will be passed at least 10000
documents to tally facet information from (unless fewer than 10000 documents
match the query, in which case they will see all of them).  Setting this to
``xapian_server.database_info(Server, document_count)`` will make the facet
counts exact, but Xapian will have to do more work for most queries so
searches will be slower.

The spy objects now contain the facet information.  You can find out how
many documents they looked at by calling
``xapian_server:resource_info(Server, SpySlot0, document_count)``.  
(All the spies will have looked at the same number of documents.)  
You can read the values from, say, ``Spy0`` like this:

.. code-block:: erlang

    -record(spy_term, {value, freq}).

    Table = xapian_server:value_count_match_spy_table(Server, SpyRes, Meta),
    Records = qlc:q(Table).

Each spy can be used with ``xapian_server:match_set/2`` only once.


Restricting by Facet Values
~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you're using the facets to offer the user choices for narrowing down
their search results, you then need to be able to apply a suitable filter.

For a single value, you could use ``#x_query_value_range{}`` with the
same start and end, 
.. or ``Xapian::MatchDecider``, 
but it's probably most
efficient to also index the categories as suitably prefixed boolean terms and
use those for filtering.
