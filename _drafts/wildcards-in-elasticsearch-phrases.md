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
run queries using wildcards, the queries need to be converted to use 
[span_near](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-span-near-query.html).

A [match_phrase](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-match-query-phrase.html) 
query matches, as the name suggests, phrases.  If you search for 'the cat sat',
then you get records containing that phrase, and not 'sat the cat' or 
'the green cat sat'.  However, it does not support wildcards.

A [wildcard query](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-wildcard-query.html)
matches terms based on a wildcard pattern including the two wildcard characters `?*`

If the only wildcard is a star at the end of the phrase, then a 
[match_phrase_prefix](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-match-query-phrase-prefix.html)
 query can be used instead.

`the pheasant pl*` becomes

```json
{
  "query": {
    "match_phrase_prefix": {
      "message": {
        "query": "the pheasant pl"
      }
    }
  }
}
```

However, if there are wildcards within the query term, then span_near will be 
needed.

You can rewrite a match_phrase query as a span_near query by splitting on 
whitespace to produce a sequence of [span_terms](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-span-term-query.html)


```json
{
  "query": {
    "match_phrase": {
      "message": "the pheasant plucker"
    }
  }
}
```

becomes

```json
{
  "query": {
    "span_near": {
    	"in_order": true,
    	"slop": 0,
    	"clauses": [
	    	{ "span_term": { "message": "the" } },
	    	{ "span_term": { "message": "pheasant" } },
	    	{ "span_term": { "message": "plucker" } }
    	]
    }
  }
}
```

In the span_near, having `slop=0` and `in_order=true` makes it behave like a 
phrase.

Now that the search term has been broken up and turned into individual
subqueries, any of the tokens that contain wildcard characters need to be
turned into [span_multi](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-span-multi-term-query.html)
queries. This allows the individual clause to contain a [term level query](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/term-level-queries.html)
such as a wildcard query.

so `the p?easant plucker` becomes

```json
{
  "query": {
    "span_near": {
    	"in_order": true,
    	"slop": 0,
    	"clauses": [
	    	{ "span_term": { "message": "the" } },
	    	{ "span_multi": 
	    		{ "match": {
	    			"wildcard": {
	    				"message": {
	    					"value": "p?easant"
	    				}
	    			}	
	    		}
	    		}
	    	},
	    	{ "span_term": { "message": "plucker" } }
    	]
    }
  }
}
```

or a prefix, so `the pheasant pluck* comes` becomes:

```json
{
  "query": {
    "span_near": {
    	"in_order": true,
    	"slop": 0,
    	"clauses": [
	    	{ "span_term": { "message": "the" } },
	    	{ "span_term": { "message": "pheasant" } },
	    	{ "span_multi": 
	    		{ "match": {
	    			"prefix": {
	    				"message": {
	    					"value": "pluck"
	    				}
	    			}	
	    		}
	    		}
	    	},
	    	{ "span_term": { "message": "comes" } }
    	]
    }
  }
}
```

The trouble is that, as it says in the Elasticsearch docs, 

	span_multi queries will hit too many clauses failure if the number of terms
	that match the query exceeds the boolean query limit (defaults to 1024).

So the next thing to do is to set [rewrite=top_terms_N](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-multi-term-rewrite.html)
appropriately. This depends on the individual application. It will need to 
be set it to a sufficiently high number that it actually returns good results.

```json
{
  "query": {
    "span_near": {
    	"in_order": true,
    	"slop": 0,
    	"clauses": [
	    	{ "span_term": { "message": "the" } },
	    	{ "span_multi": 
	    		{ "match": {
	    			"wildcard": {
	    				"message": {
	    					"value": "p?easant",
	    					"rewrite": "top_terms_10000"
	    				}
	    			}	
	    		}
	    		}
	    	},
	    	{ "span_term": { "message": "plucker" } }
    	]
    }
  }
}
```


All of that covers the case of wildcards within tokens.  There is another case
to look out for: the * wildcard as a token on its own.

Using a * wildcard to match a term in a span_multi is bad for performance, and
will probably not work very well due to the rewrite behaviour (see the [note 
about using match_phrase for autocompetion](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-match-query-phrase-prefix.html#match-phrase-prefix-autocomplete)
for a hint about why.


The best way to deal with this is use the slop facility to handle it by nesting
span_near queries accordingly.  This way, the search can keep matching the
exact phrase parts in the right sequence, but possibly split by an unknown 
token. 

"the casbah * a hurricane" becomes:

```json
{
  "query": {
	"span_near": {
		"clauses": [
			{
				"span_near": {
					"clauses": [
						{"span_term": {"my_field": "the"}},
						{"span_term": {"my_field": "casbah"}}
					],
					"in_order": true,
					"slop": 0
				}
			},
			{
				"span_near": {
					"clauses": [
						{"span_term": {"my_field": "a"}},
						{"span_term": {"my_field": "hurricane"}}
					],
					"in_order": true,
					"slop": 0
				}
			}
		],
		"in_order": true,
		"slop": 1
	}
  }
}
```

Multiple sequential * tokens translate to a higher slop value, 
so "the * * a hurricane" becomes:

```json
{
  "query": {
	"span_near": {
		"clauses": [
			{"span_term": {"my_field": "the"}},
			{
				"span_near": {
					"clauses": [
						{"span_term": {"my_field": "a"}},
						{"span_term": {"my_field": "hurricane"}}
					],
					"in_order": true,
					"slop": 0
				}
			}
		],
		"in_order": true,
		"slop": 2
	}
  }
}
```

Note that this slop is a maximum distance. If the users are expecting a * to 
mean that an unknown token definitely exists between the two parts of the 
query, then some other technique will be needed. The above query would match
"the casbah like a hurricane", or just "the a hurricane".


Multiple * tokens separated by other tokens needs to be translated into further 
nested `span_near` queries.

#TODO Example of that
 

TODO: run profile and explain.
TODO: case normalisation

TODO: other ideas