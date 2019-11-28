---
layout: post
title: Asciifolding and unicode normalisation
categories: python, elasticsearch
---

I was working on simplifying some [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch)
queries, and I discovered that asciifolding was not working on a particular field.
However, it was not so simple. Not only could I not find records with accented characters using their folded version
in a query, I could not find them with the accented version in the query either.

First, I checked that the analyzers wer working as expected, using the 
[analyze API](https://www.elastic.co/guide/en/elasticsearch/reference/current/_testing_analyzers.html).

```
curl -X GET "localhost:9200/my_sample_index/_analyze?pretty" -H 'Content-Type: application/json' -d$'
{
  "field": "my_field",
  "text" : "rosé wine"
}
'
{
  "tokens" : [
    {
      "token" : "rose",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "wine",
      "start_offset" : 5,
      "end_offset" : 9,
      "type" : "<ALPHANUM>",
      "position" : 1
    }
  ]
}
```

That seemed to work.  The token _rosé_ became _rose_, as expected.

I added a record _rosé wine_ using cURL directly into Elasticsearch...
```
curl -X POST "localhost:9200/my_sample_index/good" -H 'Content-Type: application/json' -d$'
{
  "my_field": "rosé wine"
}'
```
... and successfully found it with a query for _rose_
```
curl -X POST "localhost:9200/my_sample_index -H 'Content-Type: application/json' -d$'
{
	"query": {
		"term": {
			"my_field": "rose"
		}
	}
}'
```
                  
Using the application I was working on, I added two records, _rose wine_ and _rosé wine_.
Here is where it became really odd.
A search for _rose_, predictably returned _rose wine_.  However, a search for _rosé_
also returned _rose wine_.  It was as though the accented version had simply been discarded.

The record was definitely present. '/_search?pretty' returned both of them.

The problem, it turned out was [unicode normalization](https://unicode.org/reports/tr15/).

The application that inserted the records was first normalizing the unicode to a _decomposed_
form.  The `asciifolding` token filter only operates on composed forms.  What was actually
happening, was that _rosé_ (decomposed) was being indexed as _rosé_, but when I tried to query
it, the search term was being successfully folded to _rose_.

There are two possible solutions to this problem. Which to choose depends on context.

1. Only provide _composed_ forms (NFC/NFKC) to Elasticsearch
2. Use the [ICU Analysis Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-icu.html)

If you cannot install plugins to your Elasticsearch instance, then you will have to go with
option 1.  If your application is going to have to work with a wider range of languages
using non-latin characters, then you are likely to want to use the ICU plugin everywhere
for consistency.

```python

import unicodedata
from elasticsearch import Elasticsearch

es = Elasticsearch()

config = {
  "settings": {
    "analysis": {
        "analyzer": {
            "ascii_folding_analyzer": {
              "tokenizer": "standard",
              "filter": [
                "asciifolding",
              ]
            },
            "icu_folded_analyzer": {
                "tokenizer": "icu_tokenizer",
                "filter": [
                  "icu_folding"
                ]
            }
        }
    }
    },
    "mappings": {
    "lexicalEntry": {
      "properties": {
        "ascii_folded_field": {
            "type": "text",
            "analyzer": "ascii_folding_analyzer"
          },
          "icu_folded_field": {
              "type": "text",
              "analyzer": "icu_folded_analyzer"
          }
      }
    }
  }
}

es.indices.create(index="folding-demo", body=config)

try:
    for form in ('NFD', 'NFKD', 'NFC', 'NFKC'):
        term = unicodedata.normalize(form, 'rosé')
        ascii_folded = es.indices.analyze(
            index="folding-demo",
            body={
                "field": "ascii_folded_field",
                "text": term
            }
        )['tokens'][0]['token']
        icu_folded = es.indices.analyze(
            index="folding-demo",
            body={
                "field": "icu_folded_field",
                "text": term
            }
        )['tokens'][0]['token']

        print("{}: ascii: {}, icu: {}".format(form, ascii_folded, icu_folded))
finally:
    es.indices.delete(index='folding-demo')
```

outputs:

```
NFD: ascii: rosé, icu: rose
NFKD: ascii: rosé, icu: rose
NFC: ascii: rose, icu: rose
NFKC: ascii: rose, icu: rose
```
