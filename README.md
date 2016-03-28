## legis-graph-spatial

Adding geospatial indexing / querying to legis-graph as part of an interactive map visualization.

- **[Neo4j](http://neo4j.com/)** graph database. This extends the [legis-graph](https://github.com/legis-graph/legis-graph) example dataset.
- **[neo4j-spatial](https://github.com/neo4j-contrib/spatial)** to enable geospatial indexing and querying
- **[Mapbox JS](https://www.mapbox.com/mapbox.js/api/v2.3.0/)** javascript mapping library 

![](http://s3.amazonaws.com/dev.assets.neo4j.com/wp-content/uploads/20160328110022/legis-graph-spatial.gif)

See this blog post for more info.

### Create spatial layer

```

POST http://localhost:7474/db/data/ext/SpatialPlugin/graphdb/addEditableLayer

Accept: application/json; charset=UTF-8

Content-Type: application/json

{
  "layer" : "geom",
  "format" : "WKT",
  "nodePropertyName" : "wkt"
}

```
**From the [neo4j-spatial documentation](http://neo4j-contrib.github.io/spatial/#rest-api-create-a-wkt-layer), create a WKT layer.**


### Add nodes to the spatial layer

```
Example request

POST http://localhost:7474/db/data/ext/SpatialPlugin/graphdb/addNodeToLayer

Accept: application/json; charset=UTF-8

Content-Type: application/json

{
  "layer" : "geom",
  "node" : "http://localhost:7575/db/data/node/54"
}

```
**From the [neo4j-spatial documentation](http://neo4j-contrib.github.io/spatial/#rest-api-add-a-node-to-the-spatial-index), add a node to the layer.**

~~~ python

import requests
from py2neo import Graph

# A Neo4j instance with Legis-Graph
graph = Graph("http://52.70.212.93/db/data")
baseURI = "http://52.70.212.93"

# this function will add a node to a spatial layer
def addNodeToLayer(layer, nodeId):
    addNodeToLayerParams = {"node": baseURI+ "/db/data/node/" + str(nodeId), "layer": layer}
    r = requests.post(baseURI + "/db/data/ext/SpatialPlugin/graphdb/addNodeToLayer", json=addNodeToLayerParams)

# Find District nodes that have wkt property and are not part of the spatial index.
# Add these nodes to the layer
getIdsQuery = "MATCH (n:District) WHERE has(n.wkt) AND NOT (n)-[:RTREE_REFERENCE]-() RETURN id(n) AS n"
results = graph.cypher.execute(getIdsQuery)
for record in results:
    nodeId = record.n
    addNodeToLayer("geom", nodeId)

~~~

**This Python snippet queries the graph for nodes that have not yet been added to the spatial index and makes a REST request to add them to the index.**

### Spatial queries

```
Example request

POST http://localhost:7474/db/data/ext/SpatialPlugin/graphdb/findGeometriesWithinDistance

Accept: application/json; charset=UTF-8

Content-Type: application/json

{
  "layer" : "geom",
  "pointX" : 15.0,
  "pointY" : 60.0,
  "distanceInKm" : 100
}
```
**From the [neo4j-spatial documentation](http://neo4j-contrib.github.io/spatial/#rest-api-find-geometries-within--distance), finding geometries within distance of a point**

### Working with Mapbox

~~~ javascript

L.mapbox.accessToken = MB_API_TOKEN;
var map = L.mapbox.map('map', 'mapbox.streets')
  .setView([39.8282, -98.5795], 5);

map.on('click', function(e) {
  clearMap(map);
  getClosestDistrict(e);
});

~~~

**Create the map and define a click handler for the map.**

~~~ javascript

/**
  *  Find the District for a given latlng.
  *  Find the representative, commitees and subjects for that rep.
  */
function infoDistrictWithinDistance(latlng, distance) {

  var districtParams = {
    "layer": "geom",
    "pointX": latlng.lng,
    "pointY": latlng.lat,
    "distanceInKm": distance
  };

 var districtURL = baseURI + findGeometriesPath;
 makePOSTRequest(districtURL, districtParams, function (error, data) {

   if (error) {
    console.log("Error");
   } else {
    console.log(data);

   var params = {
    "state": data[0]["data"]["state"],
    "district": data[0]["data"]["district"]
   };

   var points = parseWKTPolygon(data[0]["data"]["wkt"]);

   makeCypherRequest([{"statement": subjectsQuery, "parameters": params}], function (error, data) {

    if (error) {
      console.log("Error");
    } else {
      console.log(data);

      var districtInfo = data["results"][0]["data"][0]["row"][0];
      districtInfo["points"] = points;
      districtInfo["state"] = params["state"];
      districtInfo["district"] = params["district"];
      console.log(districtInfo);

      addDistrictToMap(districtInfo, latlng);
    }
   });
 }
});

~~~


#### Parse WKT into an array of points

~~~ javascript

/**
 *  Converts Polygon WKT string to an array of [x,y] points
 */
function parseWKTPolygon(wkt) {
  var pointArr = [];
  var points = wkt.slice(10, -3).split(",");

  $.each(points, function(i,v) {
    var point = $.trim(v).split(" ");
    var xy = [Number(point[1]), Number(point[0])];
    pointArr.push(xy)
  });

  return pointArr;
}

~~~

