---
title:  "Building faceted search with elasticsearch for e-commerce: part 2"
date:   2019-08-05
<!-- tags: elasticsearch -->
excerpt: It's time to improve our aggregations to avoid repetition and rework.
comments: true
---

Before you continue, make sure you read the previous post: [Building faceted search with elasticsearch for e-commerce: part 1]({{site.url}}{{site.baseurl}}/building-faceted-search-with-elasticsearch-for-e-commerce-part-1)

Alright, let's see how we can make our queries better. Before the solution, I'd like to dive a bit into aggregation to make the understanding clearer.

## Pipeline Aggregations

By elastic.co definition:

> Pipeline aggregations work on the outputs produced from other aggregations rather than from document sets, adding information to the output tree.

Using our last dataset, lets see how it works:

{% highlight json %}
POST product/_search
{
  "aggs": {
    "group_by_brand": {
      "terms": { "field": "brand.keyword" },
      "aggs": {
        "group_by_color": {
          "terms": { "field": "color.keyword" }
        }
      }
    }
  }
}
{% endhighlight %}

As you can see, `group_by_color` is inside of `group_by_brand` aggregation, which means that this query will give us the colors grouped by each brand, the result is:

{% highlight json %}
{
  "took" : 11,
  "timed_out" : false,
  "_shards" : { ... },
  "hits" : { ... },
  "aggregations" : {
    "group_by_brand" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "hokey",
          "doc_count" : 4,
          "group_by_color" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "white",
                "doc_count" : 3
              },
              {
                "key" : "yellow",
                "doc_count" : 1
              }
            ]
          }
        },
        {
          "key" : "ubest",
          "doc_count" : 4,
          "group_by_color" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "green",
                "doc_count" : 2
              },
              {
                "key" : "black",
                "doc_count" : 1
              },
              {
                "key" : "yellow",
                "doc_count" : 1
              }
            ]
          }
        },
        {
          "key" : "beert",
          "doc_count" : 2,
          "group_by_color" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "black",
                "doc_count" : 1
              },
              {
                "key" : "white",
                "doc_count" : 1
              }
            ]
          }
        }
      ]
    }
  }
}
{% endhighlight %}


This is one of the techniques that will help us improve the query of our last post, you will understand soon. Now, as I mentioned in the last post, lets see how we will map our index.

## Index mapping

### Cleanup

First, lets remove our last index:

{% highlight json %}
DELETE product
{% endhighlight %}

### The mapping

Index mapping is the way we tell elasticsearch how to index each field (keyword, date, etc...). Our index mapping will be it:

{% highlight json %}
PUT /product
{
  "mappings": {
    "properties": {
      "string_facet": {
        "type": "nested",
        "properties": {
          "facet_name": { "type": "keyword"},
          "facet_value": { "type": "keyword"}
        }
      }
    }
  }
}
{% endhighlight %}

We have two properties: __facet_name__ (brand, color and department) and __facet_value__ (white, yellow, adventure, soccer, etc...).

Both of them has the type `keyword`. Terms aggregations must be a field of type keyword.

The `"type": "nested"` is very important here. Our documents will be inserted like this:

{% highlight json %}
{
  "string_facet":
    [
      {"facet_name":"brand","facet_value":"ubest"},
      {"facet_name":"color","facet_value":"green"},
      {"facet_name":"department","facet_value":"soccer"}
    ]
}
{% endhighlight %}

Without the type `nested`, ES would transform that document into this:
{% highlight json %}
{
  "string_facet.facet_name": ["brand", "color", "department"],
  "string_facet.facet_value": ["ubest", "green", "soccer"]
}
{% endhighlight %}

And then we would lose the relation between `brand: ubest`, `color: green` and `department: soccer. It would be possible, for example, to filter by facet_name: brand and facet_value: green, which doesn't make much sense.

With nested type, the independence of the objects is maintained.

## Populating / Querying

Lets populate our index for the new version:

{% highlight json %}
POST /_bulk
{ "index": { "_index": "product", "_id": 1} }
{"name":"Lila Macejkovic","string_facet":[{"facet_name":"brand","facet_value":"ubest"},{"facet_name":"color","facet_value":"green"},{"facet_name":"department","facet_value":"soccer"}]}
{ "index": { "_index": "product", "_id": 2} }
{"name":"Soledad O'Conner","string_facet":[{"facet_name":"brand","facet_value":"ubest"},{"facet_name":"color","facet_value":"green"},{"facet_name":"department","facet_value":"adventure"}]}
{ "index": { "_index": "product", "_id": 3} }
{"name":"Cherryl Streich","string_facet":[{"facet_name":"brand","facet_value":"beert"},{"facet_name":"color","facet_value":"white"},{"facet_name":"department","facet_value":"soccer"}]}
{ "index": { "_index": "product", "_id": 4} }
{"name":"Celinda Price","string_facet":[{"facet_name":"brand","facet_value":"ubest"},{"facet_name":"color","facet_value":"yellow"},{"facet_name":"department","facet_value":"adventure"}]}
{ "index": { "_index": "product", "_id": 5} }
{"name":"Elvin Thompson","string_facet":[{"facet_name":"brand","facet_value":"hokey"},{"facet_name":"color","facet_value":"yellow"},{"facet_name":"department","facet_value":"adventure"}]}
{ "index": { "_index": "product", "_id": 6} }
{"name":"Jenni MacGyver","string_facet":[{"facet_name":"brand","facet_value":"beert"},{"facet_name":"color","facet_value":"black"},{"facet_name":"department","facet_value":"casual"}]}
{ "index": { "_index": "product", "_id": 7} }
{"name":"Erwin Nader","string_facet":[{"facet_name":"brand","facet_value":"hokey"},{"facet_name":"color","facet_value":"white"},{"facet_name":"department","facet_value":"adventure"}]}
{ "index": { "_index": "product", "_id": 8} }
{"name":"Coleman Sanford","string_facet":[{"facet_name":"brand","facet_value":"ubest"},{"facet_name":"color","facet_value":"black"},{"facet_name":"department","facet_value":"casual"}]}
{ "index": { "_index": "product", "_id": 9} }
{"name":"Everett Metz","string_facet":[{"facet_name":"brand","facet_value":"hokey"},{"facet_name":"color","facet_value":"white"},{"facet_name":"department","facet_value":"soccer"}]}
{ "index": { "_index": "product", "_id": 10} }
{"name":"Leonida Oberbrunner","string_facet":[{"facet_name":"brand","facet_value":"hokey"},{"facet_name":"color","facet_value":"white"},{"facet_name":"department","facet_value":"adventure"}]}
{% endhighlight %}


And the query now is (using our knowledge of Pipeline Aggregations):

{% highlight json %}
POST /product/_search
{
  "aggs": {
    "facets": {
      "nested": {
        "path": "string_facet"
      },
      "aggs": {
        "names": {
          "terms": { "field": "string_facet.facet_name" },
          "aggs": {
            "values": {
              "terms": { "field": "string_facet.facet_value" }
            }
          }
        }
      }
    }
  }
}
{% endhighlight %}


Here we used nothing but the pipeline aggregation, `values` is inside `names`, and we also declare the path because it is a `nested type`:

{% highlight json %}
  "nested": {
    "path": "string_facet"
  },
{% endhighlight %}

The response now is:

{% highlight json %}
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : { ... },
  "hits" : { ... },
  "aggregations" : {
    "facets" : {
      "doc_count" : 30,
      "names" : {
        "doc_count_error_upper_bound" : 0,
        "sum_other_doc_count" : 0,
        "buckets" : [
          {
            "key" : "brand",
            "doc_count" : 10,
            "values" : {
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
          },
          {
            "key" : "color",
            "doc_count" : 10,
            "values" : {
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
            }
          },
          {
            "key" : "department",
            "doc_count" : 10,
            "values" : {
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
            }
          }
        ]
      }
    }
  }
}
{% endhighlight %}


## Conclusion

Now we solved our problems, if a new property must be added, it will just be a new object in array and the query won't change. And now, it doesn't matter if the products has the same properties or not, it will work, all we need to do is index using `facet_name` and `facet_value.

We are good by now, but it is not enough, in the next chapter we will see how that query will be if the customer filters something. See you!
