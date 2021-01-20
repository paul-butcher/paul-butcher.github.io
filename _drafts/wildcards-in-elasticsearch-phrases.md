---
layout: post
title: Wildcards in Elasticsearch Phrases
---

A few times now, I have been asked to add wildcards to an exact phrase 
(match_phrase) search in an application based on Elasticsearch.
It is not a simple matter of enabling a feature, but requires a different
kind of query, which can become quite complex.

The first thing to do is to find out the actual user story behind this
requirement. On more than one occasion, I have been asked to add wildcards
because someone wanted to write queries that match stems on a field that has
not been analysed with a language analyser.  e.g. they wanted a query like
`the cat* sat on the mat` to match either "the cat sat on the mat" or
"the cats sat on the mat".

In that scenario, the real solution is to implement better analysis on the
fields being queried (if available in the appropriate language).

If that is not the case, and the requirement is genuinely to be able to 
run queries using wildcards, you need to convert the queries to use 
[span_near](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl-span-near-query.html).



