---
title:  "Building faceted search with elasticsearch for e-commerce: part 4"
date:   2019-08-12
<!-- tags: elasticsearch -->
excerpt: In this chapter we'll see a solution to make the aggregations work as expected for an ecommerce.
comments: true
---

Before you continue, make sure you read the previous posts:
- [Building faceted search with elasticsearch for e-commerce: part 1]({{site.url}}{{site.baseurl}}/building-faceted-search-with-elasticsearch-for-e-commerce-part-1)
- [Building faceted search with elasticsearch for e-commerce: part 2]({{site.url}}{{site.baseurl}}/building-faceted-search-with-elasticsearch-for-e-commerce-part-2)
- [Building faceted search with elasticsearch for e-commerce: part 3]({{site.url}}{{site.baseurl}}/building-faceted-search-with-elasticsearch-for-e-commerce-part-3)

In this chapter we'll see a solution to make the aggregations work as expected for an ecommerce.

## Theory
To achieve this goal, we will need to create one specific aggregation for each facet filtered with all other filters except itself, example:

Filtering by `brand=hokey` and `color=white`. We will need one aggregation for brands filtered just by color white and another aggregation for colors just filtered by brand hokey. Why that?

- I want to show all brands options in case the customer also wants to filter by another brand, but we can't forget that he also choose color white;
- I want to show all colors options in case the customer also wants to filter by another color, but we can't forget that he also choose brand hokey;

And also we need a "main" aggregation that contains all filters, because we need show `departments` options within its filters (brand hokey and color white).

I must say that the number of aggregations needed may endup with a very long query depending on how many filter the customer choose (n+1, n = number of facets filtered).

I will build the query step by step and explaning the changes until we reach the full query, lets start.

## Filter aggregations

The first thing to do is move the query in the aggregation.
First, lets see how the structure changed:

Before:
{% highlight json %}
{
  "query": { ... },
  "aggs": { ... }
}
{% endhighlight %}


Now:
{% highlight json %}
{
  "aggs": {
    "aggs_all_filters" : {
      "filter": { ... }, <---- here goes the "query"
      "aggs": { ... }    <---- and here the "aggs"
    }
  }
}
{% endhighlight %}


The `aggs_all_filters can be whatever you want, I choose this name because this aggregations will be responsible to show all aggregations not filtered, so it will be filtered by all facets choosen. The full query is it:

{% highlight json %}
POST /product/_search
{
  "aggs": {
    "aggs_all_filters": {
      "filter": {
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
      "aggs": {
        "facets": {
          "nested": {
            "path": "string_facet"
          },
          "aggs": {
            "names": {
              "terms": {
                "field": "string_facet.facet_name"
              },
              "aggs": {
                "values": {
                  "terms": {
                    "field": "string_facet.facet_value"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
{% endhighlight %}

The response:

{% highlight json %}
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : { ... },
  "hits" : { ... },
  "aggregations" : {
    "aggs_all_filters" : {
      "doc_count" : 3,
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
}

{% endhighlight %}

We can see 3 buckets, but for this one, we must ignore `brand` and `color`, we will pick just `department`:

{% highlight ruby %}
Department
(2) adventure
(1) soccer
{% endhighlight %}

Now, lets append other aggregations, now that one will be responsible to bring the brands:

{% highlight json %}
POST /product/_search
{
  "aggs": {
    "aggs_all_filters": { ... },
    "aggs_brand": {
      "filter": {
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
      "aggs": {
        "facets": {
          "nested": {
            "path": "string_facet"
          },
          "aggs": {
            "aggs_special": {
              "filter": {
                "match": {
                  "string_facet.facet_name": "brand"
                }
              },
              "aggs": {
                "names": {
                  "terms": {
                    "field": "string_facet.facet_name"
                  },
                  "aggs": {
                    "values": {
                      "terms": {
                        "field": "string_facet.facet_value"
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}

{% endhighlight %}

As you can see, we are filtering by color white, but looking for brands, the response is:

{% highlight json %}
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : { ... },
  "hits" : { ... },
  "aggregations" : {
    "aggs_brand" : {
      "doc_count" : 4,
      "facets" : {
        "doc_count" : 12,
        "aggs_special" : {
          "doc_count" : 4,
          "names" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "brand",
                "doc_count" : 4,
                "values" : {
                  "doc_count_error_upper_bound" : 0,
                  "sum_other_doc_count" : 0,
                  "buckets" : [
                    {
                      "key" : "hokey",
                      "doc_count" : 3
                    },
                    {
                      "key" : "beert",
                      "doc_count" : 1
                    }
                  ]
                }
              }
            ]
          }
        }
      }
    },
    "aggs_all_filters" : { ... }
  }
}

{% endhighlight %}

Now we have all brands available with color white, our facet navigation now is:

{% highlight ruby %}
Brand
(3) hokey
(1) beert

Department
(2) adventure
(1) soccer
{% endhighlight %}

One more aggregation, for colors:

{% highlight json %}
POST /product/_search
{
  "aggs": {
    "aggs_all_filters": { ... },
    "aggs_brand": { ... },
    "aggs_color": {
      "filter": {
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
      "aggs": {
        "facets": {
          "nested": {
            "path": "string_facet"
          },
          "aggs": {
            "aggs_special": {
              "filter": {
                "match": {
                  "string_facet.facet_name": "color"
                }
              },
              "aggs": {
                "names": {
                  "terms": {
                    "field": "string_facet.facet_name"
                  },
                  "aggs": {
                    "values": {
                      "terms": {
                        "field": "string_facet.facet_value"
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}

Response:

{
  "took" : 2,
  "timed_out" : false,
  "_shards" : { ... },
  "hits" : { ... },
  "aggregations" : {
    "aggs_brand" : { ... },
    "aggs_color" : {
      "doc_count" : 4,
      "facets" : {
        "doc_count" : 12,
        "aggs_special" : {
          "doc_count" : 4,
          "names" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "color",
                "doc_count" : 4,
                "values" : {
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
              }
            ]
          }
        }
      }
    },
    "aggs_all_filters" : { ... }
  }
}

{% endhighlight %}

Colors available with brand hokey done! Our facet navigation now is:

{% highlight ruby %}
Brand
(3) hokey
(1) beert

Color
(3) white
(1) yellow

Department
(2) adventure
(1) soccer
{% endhighlight %}


Great, ours facets is now behaving like we expected, but we removed the `query` that filter the hits and our result is returning all products.
If we add the filters in `query` params, we will filter correctly our hits but the aggregations will be wrong. So what can we do? **Post filter**!


The elasticsearch documentation defines it very well:

> The post_filter is applied to the search hits at the very end of a search request, after aggregations have already been calculated.

So all we need to do is add a `post_filter` param with all filters and we are done.

{% highlight json %}
POST /product/_search
{
  "aggs": { ... },
  "post_filter": {
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
  }
}
{% endhighlight %}

## Conclusion

The query is ready.

This last part may be confusing, but pay attention in the queries and the results and you will understand it well.

Hope it helped you in some way. Bye!
