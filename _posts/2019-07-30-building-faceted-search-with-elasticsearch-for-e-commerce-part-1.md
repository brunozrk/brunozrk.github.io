---
title:  "Building faceted search with elasticsearch for e-commerce: part 1"
date:   2019-07-30
<!-- tags: elasticsearch -->
excerpt: Elasticsearch, querying, aggregations... Let's see it in practice.
comments: true
---

We all know how important are those filters usually located in the left side of an e-commerce catalog page, we can use them to filter and navigate through the products by its specific properties.

It would look something like this:

![alt]({{site.url}}{{site.baseurl}}/images/elasticsearch/facets.png)

It is called _faceted search_, defined by [WikiPedia](https://en.wikipedia.org/wiki/Faceted_search):

> Faceted search is a technique which involves augmenting traditional search techniques with a faceted navigation system, allowing users to narrow down search results by applying multiple filters based on faceted classification of the items

I’ll be publishing a serie of articles describing how I ended up building this mechanism using elasticsearch.

## Before we start

I'll assume that you know a bit of elasticsearch, I'll try to explain all the queries, but a bit of knowledge would be good. For further information, you can access the [elasticsearch website](https://www.elastic.co/products/elasticsearch).

I'm running elasticsearch on version 7.1.0.

Tip: I used docker to run up my elasticsearch server and also kibana, which I think is good to make requests to elasticsearch. Here is my [docker-compose file]({{site.url}}{{site.baseurl}}/files/elasticsearch/docker-compose.yml) if you want.

## Use case

Our use case will be an e-commerce that sells shoes. The faceted search will be on three properties: __color__, __department__ and __brand__. In the end of this serie, we'll have an output that will enable us to build something like the previous image.

### Populating Elasticsearch

Elasticsearch can create index for us on demand, which means that we can ask it to index some documents, and if that index does not exists, ES will automatically create the index and mapping. For now, we'll use this automatic behavior, but in some point we will need map our index before insert something. So, lets put some documents in ES:

```json
POST _bulk
{ "index": { "_index": "product", "_id": 1} }
{"name":"Lila Macejkovic","brand":"ubest","color":"green","department":"soccer"}
{ "index": { "_index": "product", "_id": 2} }
{"name":"Soledad O'Conner","brand":"ubest","color":"green","department":"adventure"}
{ "index": { "_index": "product", "_id": 3} }
{"name":"Cherryl Streich","brand":"beert","color":"white","department":"soccer"}
{ "index": { "_index": "product", "_id": 4} }
{"name":"Celinda Price","brand":"ubest","color":"yellow","department":"adventure"}
{ "index": { "_index": "product", "_id": 5} }
{"name":"Elvin Thompson","brand":"hokey","color":"yellow","department":"adventure"}
{ "index": { "_index": "product", "_id": 6} }
{"name":"Jenni MacGyver","brand":"beert","color":"black","department":"casual"}
{ "index": { "_index": "product", "_id": 7} }
{"name":"Erwin Nader","brand":"hokey","color":"white","department":"adventure"}
{ "index": { "_index": "product", "_id": 8} }
{"name":"Coleman Sanford","brand":"ubest","color":"black","department":"casual"}
{ "index": { "_index": "product", "_id": 9} }
{"name":"Everett Metz","brand":"hokey","color":"white","department":"soccer"}
{ "index": { "_index": "product", "_id": 10} }
{"name":"Leonida Oberbrunner","brand":"hokey","color":"white","department":"adventure"}
```

So here we are creating 10 documents on index `product`, with name, and as mentioned before, three properties: __brand__, __color__ and __department__.
To build our faceted navigation, elasticsearch provides a tool called `aggregations` . We can compare `aggregations` with `group by` from SQL, but much more powerfull.

So, lets imagine we want to count how many shoes each brand has, we can achieve this by doing the following query (`aggs` will do the work of `aggregations`):

{% highlight json %}
POST product/_search
{
  "aggs": {
    "group_by_brand": {
      "terms": { "field": "brand.keyword" }
    }
  }
}
{% endhighlight %}


- `aggs` just tells elasticsearch that we will build an aggregation query.
- `group_by_brand` can be anything.
- `"terms": { "field": "brand.keyword" }` tells elasticsearch to build an aggregation by the brand field (dont worry with `.keyword` now).

the result is:

{% highlight json %}
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {...},
  "hits" : {...},
  "aggregations" : {
    "group_by_brand" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "hokey",
          "doc_count" : 4
        },
        {
          "key" : "ubest",
          "doc_count" : 4
        },
        {
          "key" : "beert",
          "doc_count" : 2
        }
      ]
    }
  }
}
{% endhighlight %}


Within `aggregations`.`group_by_brand`.`buckets` we have the result we expected, with that, we can build our faceted search, lets complete the query with all fields:

{% highlight json %}
POST product/_search
{
  "aggs": {
    "group_by_brand": {
      "terms": {
        "field": "brand.keyword"
      }
    },
    "group_by_color": {
      "terms": {
        "field": "color.keyword"
      }
    },
    "group_by_department": {
      "terms": {
        "field": "department.keyword"
      }
    }
  }
}

response:

{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {...},
  "hits" : {...},
  "aggregations" : {
    "group_by_department" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "adventure",
          "doc_count" : 5
        },
        {
          "key" : "soccer",
          "doc_count" : 3
        },
        {
          "key" : "casual",
          "doc_count" : 2
        }
      ]
    },
    "group_by_color" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "white",
          "doc_count" : 4
        },
        {
          "key" : "black",
          "doc_count" : 2
        },
        {
          "key" : "green",
          "doc_count" : 2
        },
        {
          "key" : "yellow",
          "doc_count" : 2
        }
      ]
    },
    "group_by_brand" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "hokey",
          "doc_count" : 4
        },
        {
          "key" : "ubest",
          "doc_count" : 4
        },
        {
          "key" : "beert",
          "doc_count" : 2
        }
      ]
    }
  }
}

{% endhighlight %}

## Conclusion

Good, we built the presentation of the faceted search, but that approach is not the ideal.
We have two poblems here:

- First: If we need remove or add a new property to that products, we would need to change our query removing or adding the new property. It is not good.
- Second: Brand, color and department works pefectly for shoes, but lets say we also want sell beers in our store (why not?). The color does not make much sense, we would need different queries for differents types of products.

In the next article, we'll see how to solve theese problems. See you.

