---
layout: post
title:  "Elasticsearch and Near Real Time Analytics"
date:   2019-09-13
categories: Elasticsearch analytics search query aggregations real-time
---
# Elasticsearch and Near Real Time Analytics

Recently, we got a request to create an analytics dashboard for the application's data we have. The data was stored within PostgreSQL and our initial idea was to build a queries which would drive these dashboards.

Soon after we started working on this, we realized that this approach might not be the most ideal one. We ended up creating special tables to drive analytics, installing plugins to support spatial queries, and writing really complex queries which were not fast enough. Alternatively,  we even had to write multiple queries in order to support a single metric.

Our second approach was to build analytics periodically, but this isn't real time or near real time so we didn't go with it.

Finally, after some research, we realised that Elasticsearch could help us achieve real time analytics.


### Elasticsearch, Search and Aggregations

First, what is Elasticsearch ?

>Elasticsearch is an open-source, RESTful, distributed search and analytics engine built on Apache Lucene.

Integration to Elasticsearch is done through easy-to-use REST API.

Elasticsearch stores data as JSON documents, where documents are stored within an index. Subsequently, we can define an index as a collection of documents.

If we compare this to the SQL world, we can say that an index is to elasticsearch what a table is to a SQL database, and a document is to an index what a record/row is to a SQL table.

Elasticsearch is schemaless, meaning we can create an index without defining the fields that the documents will have. Elasticsearch will actually, behind the scene, create schema/mapping based on the data in the index.

However, the manual provision of mapping is most desirable as we can also specify what type of analysis we want on each field.


##### Create index and insert data

As already mentioned, Elasticsearch is exposed via REST API. We can use any HTTP client to communicate with Elasticsearch or [Kibana Console](https://www.elastic.co/guide/en/kibana/current/console-kibana.html) which has some nice features as query autocomplete.


In order to create new `index` we'll execute `HTTP PUT {es_host}/{name_of_index}` with providing settings or mapping data in request body. 

Within the settings object, we define index specific settings. An example of these settings would be number_of_shards or number_of_replicas.

Within mapping, we define the schema of our documents.

Note that we don't need to provide any data in settings/mapping and defaults will be used.

For example:

```json
PUT /users
{
    "settings" : { //we define settings for our index here
        "number_of_shards" : 1 
    },
    "mappings" : {// schema. We define fields of our documents here
        "properties" : {
            "name" : { "type" : "text" },
            "date" : { "type" : "date" },
            "salary": {"type": "integer"}
        }
    }
}
```


In order to insert document into a index we'll execute `HTTP POST {host}/{name_of_index}/_doc/`

For example:

```json
POST users/_doc/
{
    "name" : "John Doe",
    "date" : "2009-11-15T14:12:12",
    "salary": 3000
}

```

A response in this case would look like:

```json
{
 "_index" : "audit",
 "_type" : "_doc",
 "_id" : "mNMGS20B_e9lX4DjI5XZ", //This is id of record we created
 "_version" : 1,
 "result" : "created",
}
```

By default, elastic will create random ID for each document but we can also provide a ID for each record with adding {id} into request: `HTTP POST {host}/{name_of_index}/_doc/{id}`

So what is the key in getting analytics data from Elasticsearch ? Well, its all about `Search` and `Aggregations`. First we'll use query to filter out only data we care about and than we'll use aggregations to collect that data into a meaningfull analytics.

##### Querying Elasticsearch

Elasticsearch provides JSON based DSL to define queries. 

To compare with SQL, this SQL query:

```sql
SELECT sum(salary) FROM users WHERE name = "Jon" LIMIT 10;
```

in Elasticsearch will look like:

```json
GET users/_search
{
	"size": 10,
	"query" : {
		"bool": {
			"must": [
				{
					"match": {
						"name": "Jon"
					}				
				}
			]
		}	
	},
	"aggs": {
		"SalarySum": {
			"sum": {
				"field": "salary"			
			}
		}	
	}
}
```

Key parts of Elasticsearch DSL would be  `query` and `aggs`.

##### Query

Within `query` in Elasticsearch search request we define the data we want to fetch. To compare with SQL, this would be place where we put all our conditions like `name = 'Jon'` or `age BETWEEN 10 AND 30`...

##### Aggs

This is place where we define our aggregations.

Elasticsearch has large specter of aggregate functions which make it easy to get all kinds of different analytics over the data set. Full documentation on Aggregations you can find on this [link](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html).

The best part of it is that due the powerfull search and flexibility of aggregations, we can use Elasticsearch to build awesome analytics engine.

### Hands On

#### Data preparation

For the blog purpose lets imagine we work for a Bookstore and our manager, Jane, requested us to provide some analytics.

Lets say our books data is stored within some relational database with tables customer, book, audit, or something like this. Our audit table would hold information about book, customer, date when book is ordered...

For example, it could look like this:

```SQL
INSERT INTO audit VALUS (book_id, customer_id, order_date, price)
```

Our first step would be to transform/denormalize the data into something more Elasticsearch friendly. 

We will create index audit with mapping:

```json
PUT audit
{
  "mappings": {
    "properties": {
        "book": {
            "properties": {
                "category": {
                    "properties": {
                        "id": {
                            "type": "long"
                        },
                        "name": {
                            "type": "text",
                            "fields": {
                                "keyword": {
                                    "type": "keyword"
                                }
                            }
                        }
                    }
                },
                "id": {
                    "type": "long"
                },
                "name": {
                    "type": "text",
                    "fields": {
                        "keyword": {
                            "type": "keyword"
                        }
                    }
                }
            }
        },
        "customer": {
            "properties": {
                "firstName": {
                    "type": "text",
                    "fields": {
                        "keyword": {
                            "type": "keyword"
                        }
                    }
                },
                "id": {
                    "type": "long"
                },
                "lastName": {
                    "type": "text",
                    "fields": {
                        "keyword": {
                            "type": "keyword"
                        }
                    }
                }
            }
        },
        "price": {
            "type": "long"
        },
        "time": {
            "type": "date"
        }
    }
}
}
```

so our documents in elastic would look like:

```json
POST audit/_doc/
{
	"book": {
		"id": 15,
		"name": "JavaScript: The Good Parts",
		"category": {
		    "id": 5,
			"name": "Software"
	  	}
	},
	"customer": {
        "id": 12,
		"firstName": "John",
		"lastName": "Doe"
	},
	"price": 50,
	"time": 1510777312961
}
```

Note that, unlike SQL in elasticsearch we prefer to denormalize data and store all the information we can within a document. This would allow us to be flexible enough and to perform fast queries. It is possible to setup relations between documents in an index but this will affect our performance.

However, feel free to use it if you find it necessary for data redundancy. In our case, it's not likely that a book name, consumer name will change so we'll keep it like this.


#### Ready to go

OK, now we have our data indexed  so lets see what our clients want to see.


##### Jane: I want to see which books are most popular.

Ok this is easy one, we'll just get top N frequent books ( by id in our case). We can achieve that by using `terms` aggregation. For example:


```json
GET audit/_search
{
  "aggs": {
    "TopBooks": {
      "terms": {
        "field": "book.id",
        "size": 10
      }
    }
  }
}
```

[Terms](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html) aggregation is used to fetch unique values along with their count. Terms aggregation is bucket aggregation and in its response we'll get number of buckets where key of each bucket will be `book.id` in our case. As we specified `"size": 10` we'll get maximum 10 buckets in response. The example response may look like:

```json
{
    ...
    "aggregations" : {
        "TopBooks" : {
            "doc_count_error_upper_bound": 0, 
            "sum_other_doc_count": 0, 
            "buckets" : [ 
                {
                    "key" : "15", //This is book.id property from our index
                    "doc_count" : 650 //This tells us there was 650 books with book.id = 15
                },
                {
                    "key" : "642",
                    "doc_count" : 369
                },
                {
                    "key" : "1,
                    "doc_count" : 222
                }
                ... more buckets
            ]
        }
    }
}
```

#####Jane: Ok, but I want those just for last month

Sure, lets just add filter for `time` field.

```json
GET library_audit/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "time": {
              "gte": "now-1m" 
            }
          }
        }
      ]
    }
  },
  "aggs" {
   ...
   }
}
```

To filter out data by date we use [Range](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html) query. Note that now we're writting our query as part of `query` object, not `aggs`. `query` part of elastic query is used to filter out data we want. Later the filtered data is used in `aggs`. So in our case we will first filter only documents within the given range and than used the results of query in aggregations. 


##### Jane: Can we get them grouped by category ?

Well,  yes. We can use sub-aggregations for such case. Sub-aggregations allow us to aggregate data on results of previous aggregation. In our case, we will first aggregate by `category.id` to get top book categories, and than as sub aggregation of categories we'll add aggregation by `book.id`. This way we'll achieve to get top books for each category.

```json
{
  "aggs": {
    "CategoryBreakDown": {
      "terms": {
        "field": "book.category.id",
        "size": 10
      },
      "aggs": {
        "TopBooks": {
          "terms": {
            "field": "book.id",
            "size": 10
          }
        }
      }
    }
  }
}
```

We can add as many sub-aggregations as we want, as long as aggregation we use support sub-aggregations. 



##### Jane: Nice, but can we get these results for each week/months/year ?

To achieve this we'll add `date_histogram` aggregation as our root aggregation. [Date Histogram](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-datehistogram-aggregation.html) will bucket our data based on `interval` we set. So if we decide to split our data into buckets of months, we'll set `"interval": "month"` and we'll get buckets for each month. Now we can sub aggregate each month bucket with analytics of interest. In our case we'll aggregate top books for each month.

```json
{
  "aggs": {
    "PerMonth": {
      "date_histogram": {
        "field": "time",
        "interval": "month"
      },
      "aggs": {
        ...
      }
    }
  }
}
```
Example response would look like:

```json
{
    ...
    "aggregations" : {
      "PerMonth" : {
        "buckets" : [
          {
            "key_as_string" : "2019-05-01T00:00:00.000Z",
            "key" : 1556668800000,
            "doc_count" : 167,
            "TopBooks" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                {
                  "key" : 5,
                  "doc_count" : 41 // this tells us that we sold 41 book in May 2019 with ID 5
                },
                {
                  "key" : 2,
                  "doc_count" : 38 
                },
                {
                  "key" : 3,
                  "doc_count" : 34
                },
                {
                  "key" : 4,
                  "doc_count" : 29
                },
                {
                  "key" : 1,
                  "doc_count" : 25
                }
              ]
            }
          },
          {
            "key_as_string" : "2019-06-01T00:00:00.000Z",
            "key" : 1559347200000,
            "doc_count" : 279,
            "TopBooks" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                ...
              ]
            }
          },
        ...
        ]
      }
    }
  }
  
```


#####Jane: Nice, lets get some statistics on book prices.

In this case we can use `stats` or `extended_stats` aggregations. These aggregations are doing multiple statistics over numeric fields for example:  `count`, `min`, `max`, `avg`, `sum`...

```json
{
  "aggs": {
    "PriceStatistics": {
      "stats": {
        "field": "price"
      }
    }
  }
}
```

where response looks like:

```json
  "aggregations" : {
    "PriceStatistics" : {
      "count" : 999,
      "min" : 5.0,
      "max" : 100.0,
      "avg" : 50.72472472472472,
      "sum" : 50674.0
    }
  }
```

Of course, it is possible to execute any of the aggregations separately.



#### A lot of more


 These are just few simple examples on how to get some analytics from Elasticsearch but I believe you get the idea.
There is much more stuff that can be done, and lot of more aggregations to explore. Just to mention a few:

- **Filter aggregation** You could actually write own filter to be a aggregation. Very usefull when you want to bucket something in few buckets but cannot achieve with other aggregations.

- **Range aggregation** You could specify ranges of interest to aggregate data. Good example would be aggregating users by age into few buckets.

- **Geo aggregations** There are number of geo aggregations that you could find interesting. For example, you could use geo distance aggregations to aggregate stores based on city center.

- Lot of other [aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html) that could fit your case. 

- Elastic also has a way of managing [relationships](https://www.elastic.co/blog/managing-relations-inside-elasticsearch). There are also ways to aggregate data using relations.

#### Conclusion

As you can see, elastic is very powerful tool for having fast and flexible analytics. We used the output of aggregations to visualize our data and produce reports for our business team.

Use the search part for limiting the data you'll aggregate. Use histogram aggregations to display trendings. Use Elasticsearch to build awesome analytics dashboards!

