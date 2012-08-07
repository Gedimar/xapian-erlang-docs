.. Copyright (C) 2007,2009,2011 Olly Betts
.. Copyright (C) 2011 Justin Finkelstein
.. Copyright (C) 2011 James Aylett


Sorting
=======

By default, Xapian orders search results by decreasing relevance score.
However, it also allows results to be ordered by other criteria, or
a combination of other criteria and relevance score.

If two or more results compare equal by the sorting criteria, then their
order is decided by their document ids.  By default, the document ids sort
in ascending order (so a lower document id is "better"), but descending
order can be chosen using ``#x_enquire.docid_order = desc``.
If you have no preference, you can tell Xapian to use whatever order is
most efficient using ``#x_enquire.docid_order = dont_care``.
It is also possible to change the way that the relevance scores are calculated
- for details, see the :ref:`document on weighting schemes and
  document scoring <weighting_scheme>` for details.

Sorting by Value
----------------

You can order documents by comparing a specified document value.  Note that the
comparison used compares the byte values in the value (i.e. it's a string sort
ignoring locale), so ``1`` < ``10`` < ``2``.  If you want to encode the value
such that it sorts numerically, use a typed value slot. For example, for slot
with name "price", use:

.. code-block:: erlang
    Price = 600.5,
    %% Pass #x_value_name as a parameter
    {ok, Server} = xapian_server:open(Path, 
        [#x_value_name{name = price, slot = 0, type = float}|Params]),
    %% Use a named slot. Pass Price as a float.
    Doc = [ #x_value{slot = price, value = Price} ],
    xapian_server:add_document(Server, Doc).

You can sort by value with the next code:

.. code-block:: erlang
    #x_enquire{order = #x_sort_order{type = Type, slot = Slot}

.. see xapian_type:x_order_type().

The ``#x_sort_order.type`` field defines the order algorithm. The valid values
are:

 * ``value`` specifies the relevance doesn't affect the
   ordering at all.
 * ``value_relevance`` specifies that relevance is
   used for ordering any groups of documents for which the value is the same.
 * ``relevance_value`` specifies that documents are
   ordered by relevance, and the value is only used to order groups of documents
   with identical relevance values (note: the weight has to be exactly the same
   for values to determine the order, so this method isn't very useful when
   using BM25 with the default parameters, as that will rarely give identical
   scores to different documents).

We'll use the states dataset to demonstrate this, and the code from
dealing with dates in the :ref:`range queries <range_queries>` HOWTO::

    $ ./bin/index_ranges2.escript priv/states.csv priv/test_db/ranges2

This has three document values: slot 1 has the year of admission to
the union, slot 2 the full date (as "YYYYMMDD"), and slot 3 the latest
population estimate. So if we want to sort by year of entry to the
union and then within that by relevance, we want to add the following:

.. code-block:: erlang
    #x_enquire{value = Query,                               
               order = #x_sort_order{type = value_relevance,
                                    value = admitted_year}}.


The `#x_sort_order.reverse` parameter is `False` for ascending order, 
`True` for descending. We can then run sorted searches like this::

    $ ./bin/search_sorting.escript priv/test_db/ranges spanish
    1: #019            State of Texas       18451229          25145561
    2: #004          State of Montana       18891108            989415


Generated Sort Keys
-------------------

To allow more elaborate sorting schemes, Xapian allows you to provide a
functor object subclassed from ``Xapian::KeyMaker`` which generates a sort
key for each matching document which is under consideration.  This is
called at most once for each document, and then the generated sort keys are
ordered by comparing byte values (i.e. with a string sort ignoring locale).

Sorting by Multiple Values
~~~~~~~~~~~~~~~~~~~~~~~~~~

The call of the ``xapian_resource:multi_value_key_maker`` function returns a 
resource of the standard class ``Xapian::MultiValueKeyMaker``.
This allows to sort on more than one document value (so the first document value
specified determines the order; amongst groups of documents where that's
the same, the second document value determines the order, and so on).

We'll use this to change our sorted search above to order by year of
entry to the union and then by decreasing population.

The second parameter is a list of value names or slots.
If you want to reverse the value's order, put it inside the ``{reverse, Slot}``
tuple::

    $ ./bin/search_sorting2.escript priv/test_db/ranges2 State
     1: #041   Commonwealth of Pennsylva   17871212     12702379
     2: #044   State of New Jersey         17871218      8791894
     3: #050   State of Delaware           17871207       897934
     4: #042   State of New York           17880726     19378102
     5: #035   State of Georgia            17880102      9687653
     6: #039   Commonwealth of Virginia    17880625      8001024
     7: #047   Commonwealth of Massachus   17880206      6547629
     8: #051   State of Maryland           17880428      5773552
     9: #037   State of South Carolina     17880523      4625384
    10: #049   State of Connecticut        17880109      3574097

