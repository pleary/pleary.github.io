---
layout: post
title:  "Mapping and Elasticsearch"
comments: true
date:   2015-04-07 11:37:45
---
PostgreSQL and PostGIS are the defacto standard for mapping databases. "But Elasticsearch is webscale" I hear you saying. I'm with you, database romanticist. I couldn't find a ton of geospatial projects based on Elasticsearch, so I did some research and decided to post a little of what helped me get started. This post explores some of the basics storing and querying geospatial data in Elasticsearch.

[Elasticsearch](http://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html) is very easy to [install](https://www.elastic.co/downloads/elasticsearch). Make sure you have a working install and that you can view [http://localhost:9200/](http://localhost:9200/).

We can start from a fresh index:

{% highlight ruby %}
curl -XPOST localhost:9200/geospatial -d '{
  settings: {
    number_of_shards: 1,
    number_of_replicas: 1
  },
  mappings: {
    point: {
      properties: {
        id: { type: "integer" },
        name: { type: "string" },
        location: {
          type: "geo_point",
          lat_lon: true,
          geohash: true,
          geohash_precision:10
        },
        geojson: { type: "geo_shape" }
      }
    }
  }
}'
{% endhighlight %}

This will create a new index named geospatial, which will contain documents of type point with four properties, id, name, location, and geojson. But first, there are some other settings worth noting - [shards](http://www.elastic.co/guide/en/elasticsearch/guide/master/shard-scale.html) and [replicas](http://www.elastic.co/guide/en/elasticsearch/guide/master/replica-shards.html). These are awesome features that you should totally check out, after finishing this post. You'll also want to get familiar with the [Elasticsearch APIs](http://www.elastic.co/guide/en/elasticsearch/reference/1.x/docs-index_.html). For example let's take a look at the index we just created [http://localhost:9200/geospatial/](http://localhost:9200/geospatial/).

Back to the fields. ID and name are pretty standard, boring really. Next is a [geo_point](http://www.elastic.co/guide/en/elasticsearch/reference/1.3/mapping-geo-point-type.html), which enables lat_lon indexing for faster geo distance and bounding box filters, and [geohash](http://www.elastic.co/guide/en/elasticsearch/guide/master/geohashes.html) indexing for fast grid and lat/lon/zoom searches. Lastly there is a [geo_shape](http://www.elastic.co/guide/en/elasticsearch/reference/1.3/mapping-geo-shape-type.html), which can use [geo_shape filters](http://www.elastic.co/guide/en/elasticsearch/reference/1.4/query-dsl-geo-shape-filter.html) to compare with other indexed shapes, for example complex polygons like countries. While similar, there are some features, such as geo_shape filters, which apply to only points or shapes. For this demonstration we are using both and storing different representations of the same data in each field. 

In fact, let's make a complex polygon now. We'll start by creating a new index for places:

{% highlight ruby %}
curl -XPOST localhost:9200/places -d '{
  mappings: {
    place: {
      properties: {
        id: { type: "integer" },
        geojson: {
          type: "geo_shape"
        }
      }
    }
  }
}'
{% endhighlight %}

We can now post a new document representing a place:

{% highlight ruby %}
curl -XPOST localhost:9200/places/place/1 -d '{
  "id": 1,
  "geojson": {
    "type": "Polygon",
    "coordinates": [[[-1,51], [-1,52], [-2,52], [-2,51], [-1,51]]]
  }
}'
{% endhighlight %}

And now create two points, one that falls inside the place and another that falls outside:

{% highlight ruby %}
curl -XPOST localhost:9200/geospatial/point/1 -d '{
  "id": 1,
  "name": "Gamehenge",
  "location": "51.178,-1.826",
  "geojson": {
    "type": "Point",
    "coordinates": [-1.826,51.178]
  }
}'

curl -XPOST localhost:9200/geospatial/point/2 -d '{
  "id": 2,
  "name": "Pyramids",
  "location": "29.975,31.135",
  "geojson": {
    "type": "Point",
    "coordinates": [31.135,29.975]
  }
}'
{% endhighlight %}

Great, so now we can store points and polygons. Notice above the different formats for geo_point and geo_shape which uses GeoJSON. GeoJSON expects lon, lat but geo_point expect lat, lon. Unfortunate, confusing, and important.

Time for some querying. Elasticsearch can [query bounding boxes](http://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-bounding-box-filter.html):

{% highlight ruby %}
curl -XGET localhost:9200/geospatial/_search -d '{
  filter: {
    geo_bounding_box: {
      location: {
        bottom_left: [ -1.9, 51.1],
        top_right: [ -1.8, 51.2]
      }
    }
  }
}'
{% endhighlight %}

It can also do [geo distance](http://www.elastic.co/guide/en/elasticsearch/reference/1.x/query-dsl-geo-distance-filter.html) queries:

{% highlight ruby %}
curl -XGET localhost:9200/geospatial/_search -d '{
  filter: {
    geo_distance: {
      distance: "100km",
      location: "51,-2"
    }
  }
}'
{% endhighlight %}

And we can [compare the geo_shape](http://www.elastic.co/guide/en/elasticsearch/reference/1.x/query-dsl-geo-shape-filter.html#_pre_indexed_shape) fields with our stored polygons to find all points within a place:

{% highlight ruby %}
curl -XGET localhost:9200/geospatial/_search -d '{
  filter: {
    geo_shape: {
      geojson: {
        indexed_shape: {
          id: 1,
          type: "place",
          index: "places",
          path: "geojson"
        }
      }   
    }
  }
}'
{% endhighlight %}

Or you can just make a [polygon on the fly](http://www.elastic.co/guide/en/elasticsearch/reference/1.x/query-dsl-geo-polygon-filter.html) and query that with:

{% highlight ruby %}
curl -XGET localhost:9200/geospatial/_search -d '{
  filter: {
    geo_polygon: {
      location: {
        points: [
          [-1,50],
          [-2,50],
          [-2,52]
        ]
      }   
    }
  }
}'
{% endhighlight %}

The [geohases](http://en.wikipedia.org/wiki/Geohash) that were indexed with location can be [queried directly](http://www.elastic.co/guide/en/elasticsearch/reference/1.4/query-dsl-geohash-cell-filter.html) or [used in aggregations](http://www.elastic.co/guide/en/elasticsearch/reference/1.4/search-aggregations-bucket-geohashgrid-aggregation.html) to return the geohash cells and counts for results of a query:

{% highlight ruby %}
curl -XGET localhost:9200/geospatial/_search -d '{
  aggregations: {
    geohashes: {
      geohash_grid: {
        field: "location",
        precision: 1
      }   
    }
  }
}'
{% endhighlight %}

Geohashes represent smaller and more specific areas the longer the hash is. We declared the index to have a precision of 10, so the smallest cells (longest geohashes) will represent roughly one square meter.

So far I am very impressed with ease of use, functionality, performance and more from Elasticsearch. In my initial testing, with around 1.2 million points and tens of thousands of places of various sizes and complexities, for most queries I am seeing performance at least on par with PostGIS. But some queries for points within large polygons are up to 30 times faster in Elastisearch.

Again, I haven't seen a lot of use of Elasticsearch or geohashes in geospatial web applications so please share some examples if you have them. I'd love to see what other people are up to. I will make a follow-up post describing some work I've been doing on an Elasticsearch-based map tileserver using Node.js and [node-mapnik](https://github.com/mapnik/node-mapnik).

Oh, and if you want to destroy all the indices and data created above:

{% highlight ruby %}
curl -XDELETE localhost:9200/geospatial
curl -XDELETE localhost:9200/places
{% endhighlight %}
