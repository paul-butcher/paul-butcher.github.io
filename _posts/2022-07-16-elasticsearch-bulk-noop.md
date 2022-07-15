---
layout: post
title: Elasticsearch Bulk updates with NOOP
categories: elasticsearch
date: 2022-07-13 22:33 +0100
---

Sometimes, you want to throw a bunch of data at an existing Elasticsearch
index and let the database tell you what has changed.  The magic words
are "update" and "doc_as_upsert"

To post bulk data to Elasticsearch, you use the appropriately named 
[Bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html).
The thrust of that being that you send [NDJSON](http://ndjson.org/), with pairs
of lines.  The first says what to do, and the second contains the data to do it with.
(delete is just one line, but that's irrelevant to this post)

Having sent this, Elastic will let you know what it has done.  However, there 
is a bit of a kink when it comes to finding out that it hasn't done anything.

[The Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html#bulk-api-response-body)
states that the action in the response can only be one of create, delete, index, and update,
but actually, just like the [Update API](https://www.elastic.co/guide/en/elasticsearch/reference/8.3/docs-update.html),
_bulk will perform noop detection if you ask it nicely, thus:

```json
{"update": {"_index":"banana","_id":"world"}}
{"doc": {"hello":"world"}, "doc_as_upsert": true}
```

Without doc_as_upsert=true, this couplet would fail on inserting a 
new document:

```json
{"update": {"_index":"banana","_id":"world"}}
{"doc": {"hello":"world"}}
```

Indexing updates the document regardless of whether there are any changes:

```json
{"index": {"_index":"banana","_id":"kitty"}}
{"doc": {"hello":"kitty"}}
```