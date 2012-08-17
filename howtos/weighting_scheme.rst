.. Copyright (C) 2007,2009,2011 Olly Betts

How to change how documents are scored
======================================

The document weighting schemes which Xapian includes by default are
``BM25Weight``, ``TradWeight`` and ``BoolWeight``.  The default is
BM25Weight.

The ``#x_enquire.weighting_scheme`` field defines the weighting algorithm.

BoolWeight
----------

BoolWeight assigns a weight of 0 to all documents, so the ordering is
determined solely by other factors::
    #x_enquire{weighting_scheme = xapian_resource:bool_weight()}.


    
TradWeight
----------

TradWeight implements the original probabilistic weighting formula, which
is essentially a special case of BM25: it's BM25 with k2 = 0, k3 = 0, b =
1, and min_normlen = 0, except that all the weights are scaled by a
constant factor K::
    #x_enquire{weighting_scheme = xapian_resource:trad_weight(K)}.


BM25Weight
----------

The BM25 weighting formula which Xapian uses by default has a number of
parameters.  We have picked some default parameter values which do a good job
in general.  The optimal values of these parameters depend on the data being
indexed and the type of queries being run, so you may be able to improve the
effectiveness of your search system by adjusting these values, but it's a
fiddly process to tune them so people tend not to bother.

.. todo:: Say something more useful about tuning the parameters!

See the `BM25 documentation <bm25.html>`_ for more details of BM25.

To set this weighting scheme use::
    #x_enquire{weighting_scheme = xapian_resource:bm25_weight(
        #x_bm25_weight{k1=K1, k2=K2, l3=K3, b=B, min_normlen=MinNormLen})}.
or::
    #x_enquire{weighting_scheme = 
        xapian_resource:bm25_weight(K1, K2, K3, B, MinNormLen)}.

