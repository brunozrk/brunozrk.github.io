---
title:  "Building faceted search with elasticsearch for e-commerce: part 3"
date:   2019-08-07
<!-- tags: elasticsearch -->
excerpt: Adding filters to our search and see what happens with the facets...
comments: true
---
Before you continue, make sure you read the previous posts:
- [Building faceted search with elasticsearch for e-commerce: part 1]({{site.url}}{{site.baseurl}}/building-faceted-search-with-elasticsearch-for-e-commerce-part-1)
- [Building faceted search with elasticsearch for e-commerce: part 2]({{site.url}}{{site.baseurl}}/building-faceted-search-with-elasticsearch-for-e-commerce-part-2)

The next step we'll start filtering our results, without further delays, let's see it in practice.

## Querying

Remembering our options, we have:

{% highlight ruby %}
Brand
(4) hokey
(4) ubest
(4) beert

Color
(4) white
(2) black
(2) green
(2) yellow

Department
(5) adventure
(3) soccer
(2) casual
{% endhighlight %}

Let's imagine we'd like to filter products with `brand=hokey`, we'll start using the `query` parameter to do so:

{% highlight json %}
POST /product/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "nested": {
            "path": "string_facet",
            "query": {
              "bool": {
                "filter": [
                  {
                    "term": {
                      "string_facet.facet_name": "brand"
                    }
                  },
                  {
                    "term": {
                      "string_facet.facet_value": "hokey"
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  },
  "aggs": { no changes here... }
}
{% endhighlight %}


The query is telling elasticsearch to filter products which contains the `facet_name=brand` AND `facet_value=hokey`. As we are using `nested type`, we must declare `nested` param as we did in `aggregations`. The `bool`.`filter` is an `AND` operator.

It then returned, as expected, just 4 hits:

{% highlight json %}
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : { ... },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 0.0,
    "hits" : [
      {
        "_index" : "product",
        "_type" : "_doc",
        "_id" : "5",
        "_score" : 0.0,
        "_source" : {
          "name" : "Elvin Thompson",
          "string_facet" : [
            {
              "facet_name" : "brand",
              "facet_value" : "hokey"
            },
            {
              "facet_name" : "color",
              "facet_value" : "yellow"
            },
            {
              "facet_name" : "department",
              "facet_value" : "adventure"
            }
          ]
        }
      },
      {
        "_index" : "product",
        "_type" : "_doc",
        "_id" : "7",
        "_score" : 0.0,
        "_source" : {
          "name" : "Erwin Nader",
          "string_facet" : [
            {
              "facet_name" : "brand",
              "facet_value" : "hokey"
            },
            {
              "facet_name" : "color",
              "facet_value" : "white"
            },
            {
              "facet_name" : "department",
              "facet_value" : "adventure"
            }
          ]
        }
      },
      {
        "_index" : "product",
        "_type" : "_doc",
        "_id" : "9",
        "_score" : 0.0,
        "_source" : {
          "name" : "Everett Metz",
          "string_facet" : [
            {
              "facet_name" : "brand",
              "facet_value" : "hokey"
            },
            {
              "facet_name" : "color",
              "facet_value" : "white"
            },
            {
              "facet_name" : "department",
              "facet_value" : "soccer"
            }
          ]
        }
      },
      {
        "_index" : "product",
        "_type" : "_doc",
        "_id" : "10",
        "_score" : 0.0,
        "_source" : {
          "name" : "Leonida Oberbrunner",
          "string_facet" : [
            {
              "facet_name" : "brand",
              "facet_value" : "hokey"
            },
            {
              "facet_name" : "color",
              "facet_value" : "white"
            },
            {
              "facet_name" : "department",
              "facet_value" : "adventure"
            }
          ]
        }
      }
    ]
  },
  "aggregations" : { ... }
}


{% endhighlight %}

And then, if we want to filter by `brand=hokey` AND `color=white`:

{% highlight json %}
POST /product/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "nested": {
            "path": "string_facet",
            "query": {
              "bool": {
                "filter": [
                  {
                    "term": {
                      "string_facet.facet_name": "brand"
                    }
                  },
                  {
                    "term": {
                      "string_facet.facet_value": "hokey"
                    }
                  }
                ]
              }
            }
          }
        },
        {
          "nested": {
            "path": "string_facet",
            "query": {
              "bool": {
                "filter": [
                  {
                    "term": {
                      "string_facet.facet_name": "color"
                    }
                  },
                  {
                    "term": {
                      "string_facet.facet_value": "white"
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  },
  "aggs": { no changes here... }
}

response:
{
  "took" : 10,
  "timed_out" : false,
  "_shards" : { ... },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.0,
    "hits" : [
      {
        "_index" : "product",
        "_type" : "_doc",
        "_id" : "7",
        "_score" : 0.0,
        "_source" : {
          "name" : "Erwin Nader",
          "string_facet" : [
            {
              "facet_name" : "brand",
              "facet_value" : "hokey"
            },
            {
              "facet_name" : "color",
              "facet_value" : "white"
            },
            {
              "facet_name" : "department",
              "facet_value" : "adventure"
            }
          ]
        }
      },
      {
        "_index" : "product",
        "_type" : "_doc",
        "_id" : "9",
        "_score" : 0.0,
        "_source" : {
          "name" : "Everett Metz",
          "string_facet" : [
            {
              "facet_name" : "brand",
              "facet_value" : "hokey"
            },
            {
              "facet_name" : "color",
              "facet_value" : "white"
            },
            {
              "facet_name" : "department",
              "facet_value" : "soccer"
            }
          ]
        }
      },
      {
        "_index" : "product",
        "_type" : "_doc",
        "_id" : "10",
        "_score" : 0.0,
        "_source" : {
          "name" : "Leonida Oberbrunner",
          "string_facet" : [
            {
              "facet_name" : "brand",
              "facet_value" : "hokey"
            },
            {
              "facet_name" : "color",
              "facet_value" : "white"
            },
            {
              "facet_name" : "department",
              "facet_value" : "adventure"
            }
          ]
        }
      }
    ]
  },
  "aggregations" : { ... }
}


{% endhighlight %}

Cool! The hits are filtered as expected, but... What about our aggregations? The `query` does affect it! Lets see what happened:

{% highlight json %}
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : { ... },
  "hits" : { ... },
  "aggregations" : {
    "facets" : {
      "doc_count" : 9,
      "names" : {
        "doc_count_error_upper_bound" : 0,
        "sum_other_doc_count" : 0,
        "buckets" : [
          {
            "key" : "brand",
            "doc_count" : 3,
            "values" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                {
                  "key" : "hokey",
                  "doc_count" : 3
                }
              ]
            }
          },
          {
            "key" : "color",
            "doc_count" : 3,
            "values" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                {
                  "key" : "white",
                  "doc_count" : 3
                }
              ]
            }
          },
          {
            "key" : "department",
            "doc_count" : 3,
            "values" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                {
                  "key" : "adventure",
                  "doc_count" : 2
                },
                {
                  "key" : "soccer",
                  "doc_count" : 1
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

With that query, ours option are:

{% highlight ruby %}
Brand
(3) hokey

Color
(3) white

Department
(2) adventure
(1) soccer

{% endhighlight %}

### Conclusion

It corresponds exactly with the products returned, but thinking in an e-commerce behavior, it does not look right, let's see why.

When we filtered by `brand=hokey` it excluded all other options for brand, the same happened with `color=white`. But usually, it should be possible to filter by more than one brand (eg.: hokey and beert. It is very common, by the way). In the next chapter (and last!) we'll see how to fix it.
