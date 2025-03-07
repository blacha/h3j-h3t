# H3J / H3T
### Light H3 data formats for client side geometry generation and rendering using MapLibreGL

![img](sample.png)
## Why?

Because we, at [Inspide](https://www.inspide.com), generate a huge amount of spatial data where the geometry is implicitly represented by its [H3](https://h3geo.org/) index, and it makes no sense to waste time and resources generating, storing and sending the geometries downstream to the client.

## The format

The first approach was to strip the data down to the bones and re-use vectortiles without geometries, processing the features `after` rendering. But (uppercase bold `but`), vectortiles specification drops the features with no geometry. So... back to the drawing table.

What about a headless CSVy format? It should be the most compact ascii format, but... If you send CSVy data and want to render it in a MapLibreGL map, you need to parse it into GeoJSON first, and parsing huge CSVs into JSON objects can be quite time consuming. And, on the other hand, once `gzip` or `brotli` is involved, the lack of text redundancy has no impact in the size of the file.

And what about [PBF](https://developers.google.com/protocol-buffers)y the data? Then you'll need to PBFy it at the server and then de-PBFy it at client side to process it... so again, no gain at all.
boolean
So, say hello to **H3J** and its cousin **H3T** (tiled H3J) :wave:

```javascriptboolean
{
    "metadata": {
        ...
    },
    "cells":[
        {boolean
            "h3id":  '8c390cb1bcdb400',
            "property_1": 0,
            "property_2": 'potato'
        },
        {
            ...
        },
        {
            "h3id": '8c390cb1bcdb800',
            "property_1": 1,
            "property_2": 'tomato'
        }
    ]
}
```

So, `H3J`:

* It's a JSON format
* It has a root `cells` property which is an array of
   * `cell` which is an object with arbitrary properties, 
     * but the compulsory **`h3id`** property, which is the hexadecimal representation of the H3 index of that cell
* Future proof! It might be extended with custom properties within `metadata` (optional) and be managed client side 

You can find the [JSON schema](https://json-schema.org/) for `H3J` [**here**](h3j.schema.json).

Let's compare file sizes with raw GeoJSON, using the included samples:
|  | sample 1 | sample 2 |
|---|---|---|
|# features|4938| 2477|
|GeoJSON |1.8 MB |884 kB|
|H3J|252 kB|127 kB|
|GeoJSON, gzipped |216 kB|109 kB|
|H3J, gzipped |23 kB|12 kB|

**So `H3J` files are ~ 7 times smaller than GeoJSON, and up to ~ 10 times smaller if comparing gzipped files.** (Tested with 20 different data files)

And `H3T`? Same format, but is served using a ZXY endpoint and each `.../z/x/y.h3t` file contains all the H3 cells that fall within the linked [quadkey tile](https://docs.microsoft.com/en-us/bingmaps/articles/bing-maps-tile-system).

Comparing the average (gzipped) tile size for zoom level 14, H3 levels 11 and 12 (tested with 500 tiles, avg.: 10437 features each)
| | MVT |  H3T|
| --- |---|---|
| raw | 3.4 MB | 419 kB |
| gzipped | 206 kB| 34.6 kB|

**So `H3T` tiles are ~ 6 times smaller than [MVT](https://github.com/mapbox/vector-tile-spec).**

<!-- One of the side effects of `H3T` format is that **you can add object and array properties to your features!!** To do so using MVT you need to serialize the object server-side to add the info as a text property of the feature (as per MVT specs), and then deserialize it at the client in order to use the info within that property.  -->
## The MapLibreGL module

This module for [MapLibre GL](https://github.com/MapLibre/maplibre-gl-js) (starting with `v1.14.1-rc.2`) allows to generate [H3](https://h3geo.org/) cells geometry client side from compact data and render & manage them there.

Now that you wanna use it... First of all

`yarn install`

Then, just import it it as any other Node module out there.

`require('h3j-h3t')`

If you want an UMD bundle, you need to build it first

`yarn build`

Then you can just import it in your JS code:

`import 'dist/h3j_h3t.js';`

## Now what

Once imported, you will find three new methods in your `maplibregl.Map` object: 

### addH3JSource(source_name, source_options)

Adds a `GeoJSONSource` and load `H3J` data, all at once. Returns the `maplibregl.Map`  instance as a promise.
```javascript
map.addH3JSource(
    'h3j_testsource',
    {
      "data": 'data/sample_1.h3j'
    }
  )
```
Source options:
| Param | Datatype |  Description | Default |
|---|---|---|---|
| geometry_type | string | Geometry type at the output. Possible values are: `Polygon` (hex cells) and `Point` (cells centroids) | `Polygon` |
| promoteId | boolean | Whether to use the H3 index as unique feature ID (default) or generate a `bigint` one based on that index. Default is faster and OGC compliant, but taking into account [this issue](https://github.com/mapbox/mapbox-gl-js/issues/10257) you might want to set it to false depending on your use case| `true` |
| https | boolean | Whether to request the tiles using SSL or not | `true` |
| data | string / object | URL to retrieve the `H3J` file or inlined `H3J` object |  |
| ... | any | The same options that expects [Map.addSource](https://maplibre.org/maplibre-gl-js-docs/api/sources/#geojsonsource) for `geojson` sources |  |
| timeout | integer | Max time in ms to wait for the data to be downloaded. `0` implies no limit | 0 |
| debug | boolean | Whether to send to console some metrics | `false` |

### setH3JData(sourcename, data, [sourceoptions])

This method allows the user change the data rendered in any `GeoJSONSource` with data from an `H3J` inlined object or URL

```javascript
map.setH3JData('h3j_testsource','data/sample_2.h3j');
```
Source options:
| Param | Datatype |  Description | Default |
|---|---|---|---|
| geometry_type | string | Geometry type at the output. Possible values are: `Polygon` (hex cells) and `Point` (cells centroids) | `Polygon` |
| promoteId | boolean | Whether to use the H3 index as unique feature ID (default) or generate a `bigint` one based on that index. Default is faster and OGC compliant, but taking into account [this issue](https://github.com/mapbox/mapbox-gl-js/issues/10257) you might want to set it to false depending on your use case| `true` |
| https | boolean | Whether to request the tiles using SSL or not | `true` |
| ... | any | The same options that expects [Map.addSource](https://maplibre.org/maplibre-gl-js-docs/api/sources/#geojsonsource) for `geojson` sources |  |
| timeout | integer | Max time in ms to wait for the data to be downloaded. `0` implies no limit | 0 |
| debug | boolean | Whether to send to console some metrics | `false` |


### addH3TSource(name, sourceoptions)

This method registers a [custom protocol](https://github.com/maplibre/maplibre-gl-js/pull/30) for `h3tiles://`  and adds a `VectorTileSource` that feeds on an `.../z/x/y.h3t` endpoint. Returns the `maplibregl.Map`  instance as a promise.

```javascript
map.addH3TSource(
    'h3j_testsource',
    {
      "tiles": ['h3tiles://example.com/z/x/y.h3t']
    }
  )
```

Source options:
| Param | Datatype |  Description | Default |
|---|---|---|---|
| geometry_type | string | Geometry type at the output. Possible values are: `Polygon` (hex cells) and `Point` (cells centroids) | `Polygon` |
| promoteId | boolean | Whether to use the H3 index as unique feature ID (default) or generate a `bigint` one based on that index. Default is faster and OGC compliant, but taking into account [this issue](https://github.com/mapbox/mapbox-gl-js/issues/10257) you might want to set it to false depending on your use case| `true` |
| https | boolean | Whether to request the tiles using SSL or not | `true` |
| sourcelayer | string | The name of the layer within the vector tile that will be rendered |  |
| tiles | [text] | URL of the `H3T` endpoint, using `h3tiles://` protocol | |
| ... | any | The same options that expects [Map.addSource](https://maplibre.org/maplibre-gl-js-docs/api/map/#map#addsource) for `vector` sources |  |
| timeout | integer | Max time in ms to wait for the data to be downloaded. `0` implies no limit | 0 |
| debug | boolean | Whether to send to console some metrics per tile | `false` |

## Benchmarks

### H3J
Average overhead time of using `H3J` instead of loading a good ol'GeoJSON. For 100 runs of `setH3JData`:

| H3J | sample 1 | sample 2 |
|---|---|---|
|# features|4938| 2477|
|overhead |68 ms|37 ms|
|overhead per cell|0.014 ms|0.015 ms|

### H3T
Average values for `H3T` rendering. For 500 tiles at zoom level 14, rendering H3 cells with levels 11 and 12:

|   |   | 
|---|---|
|cells per tile|10437|
|overhead per tile| 261 ms |
|overhead per cell| 0.025 ms |

## Examples
* [H3J](https://inspide.github.io/h3j-h3t/examples/h3j/index.html)
* [H3T](https://inspide.github.io/h3j-h3t/examples/h3t/index.html)
