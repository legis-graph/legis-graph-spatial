## legis-graph-spatial

Adding geospatial indexing / querying to legis-graph as part of an interactive map visualization.

- **[Neo4j](http://neo4j.com/)** graph database. This extends the [legis-graph](https://github.com/legis-graph/legis-graph) example dataset.
- **[neo4j-spatial](https://github.com/neo4j-contrib/spatial)** to enable geospatial indexing and querying
- **[Mapbox JS](https://www.mapbox.com/mapbox.js/api/v2.3.0/)** javascript mapping library 

![](http://s3.amazonaws.com/dev.assets.neo4j.com/wp-content/uploads/20160328110022/legis-graph-spatial.gif)

See [this blog post](http://www.lyonwj.com/2016/08/09/neo4j-spatial-procedures-congressional-boundaries/) for more info.

### Create spatial layer

```
call spatial.addWKTLayer('geom', 'wkt')
```
**From the [neo4j-spatial documentation](http://neo4j-contrib.github.io/spatial/#rest-api-create-a-wkt-layer), create a WKT layer.**


### Add nodes to the spatial layer

```
MATCH (d:District)
WITH collect(d) AS districts
CALL spatial.addNodes('geom', districts) YIELD node
RETURN count(*)
```

### Spatial queries

```
CALL spatial.withinDistance('geom',
  {latitude: 37.563440, longitude: -122.322265}, 1) YIELD node AS d
WITH d, d.wkt AS wkt, d.state AS state, d.district AS district LIMIT 1
MATCH (d)<-[:REPRESENTS]-(l:Legislator)
MATCH (l)-[:SERVES_ON]->(c:Committee)
MATCH (c)<-[:REFERRED_TO]-(b:Bill)
MATCH (b)-[:DEALS_WITH]->(s:Subject)
WITH wkt, state, district, l.govtrackID AS govtrackID, l.lastName AS lastName,
  l.firstName AS firstName, l.currentParty AS party, s.title AS subject,
  count(*) AS strength, collect(DISTINCT c.name) AS committees
ORDER BY strength DESC LIMIT 10
WITH wkt, state, district, {lastName: lastName, firstName: firstName,
  govtrackID: govtrackID, party: party, committees: committees} AS legislator,
  collect({subject: subject, strength: strength}) AS subjects
RETURN {legislator: legislator, subjects: subjects, state: state,
  district: district} AS r, wkt LIMIT 1
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

