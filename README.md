# SPARQLing Unicorn QGIS Plugin

This plugin adds a GeoJSON layer from SPARQL enpoint queries.

The necessary python libs can be found here: https://github.com/sparqlunicorn/unicornQGISdepInstaller.

qgisMinimumVersion = 3.0

Doxygen Documentation https://sparqlunicorn.github.io/sparqlunicornGoesGIS/

## QGIS Plugin

This Plugin is listed under die experimentail QGIS Pluigins:

* https://plugins.qgis.org/plugins/sparqlunicorn/

## Credits

* developer: SPARQL Unicorn, Florian Thiery, Timo Homburg
* contact: qgisplugin@sparqlunicorn.link

# Documentation and Help

This documentation should help users and developers to get a better understanding about the internals of the SPARQL Unicorn QGIS plugin.

## Querying geospatial data

The SPARQL Unicorn plugin returns QGIS layers from specifically formatted SPARQL queries. 
In this section the kinds of queries which are supported are presented.

### SPARQL queries including a geometry literal

The SPARQL queries need to include the following components:

* A query variable which is used to return the geometry literals (usually *?geo*)
* A query variable indicating the URI of the owl:Individual which is queried (i.e. the feature id) (usually *?item*)

Example:

    SELECT ?item ?geo WHERE {
       ?item a ex:House .
       ?item geosparql:hasGeometry ?geom_obj .
       ?geom_obj geosparql:asWKT ?geo .
    } LIMIT 10

This query queries fictional houses from an unspecified SPARQL endpoint. Each house is associated with a URI which is captured in the *?item* variable.
The geometry literal (here a WKTLiteral) is captured in the *?geo* variable.
The results of any additional query variables become new columns of the result set, i.e. the QGIS vector layer.

The SPARQL Unicorn QGIS plugin currently supports the parsing of the following literal types:

* OGC GeoSPARQL WKT Literals
* GeoJSON Literals

In the case that the triple store does not include geometry literals but instead provides two properties with latitude and longitude, two variables *?lat* *?lon* have to be included in the query description.

Example using the [Kerameikos](http://kerameikos.org/) Triple Store:

    SELECT ?item ?lat ?lon WHERE {
         ?item a <http://www.cidoc-crm.org/cidoc-crm/E53_Place>.
         ?item wgs84_pos:lat ?lat .
         ?item wgs84_pos:long ?lon . 
    } LIMIT 10
    
The triple store configuration should reflect if geometry literals are used or if lat/lon properties are provided.

### Querying all properties of a given semantic class

Very often, a SPARQL query is used to discover linked data, so that not all properties of a given class which should be returned are known.
Similarly, one typically does not want to specify a query variable for all columns of the QGIS vector layer if this can be avoided.

Example: Query 100 schools from Wikidata with all properties

    SELECT ?item ?itemLabel ?rel ?val ?geo WHERE {
        ?item wdt:P31 wd:Q3914 .
        ?item wdt:P625 ?geo .
        ?item ?rel ?val .
        SERVICE wikibase:label { bd:serviceParam wikibase:language "[AUTO_LANGUAGE],en". }
    } LIMIT 100

This query uses the special variables *?rel* and *?val* to indicate that all relations and values of the school instances should be included in the result set.

### SPARQL queries without geometry literals

SPARQL queries without geometry literals may be issued to a triple store when the appropriate checkbox ("Allow non-geo queries") is selected in the user interface. This allows for the executition of arbitrary SPARQL queries and returns a QGIS layer without an attached geometry which might be used for merging with other QGIS layers.

Example:

    SELECT ?tower WHERE {
     ?tower a <http://onto.squirrel.link/ontology#Watchtower>.
    } LIMIT 100

### Querying instances with the help of data included in other QGIS layers

The columns of a loaded QGIS vector layer may be used as a query input in the SPARQL Unicorn QGIS plugin.

To achieve this behavior QGIS columns are converted to a SPARQL values statement as illustrated in the following example.

Consider a QGIS vector layer of houses which is formatted as follows:

| Geometry   | Address |
|---|---|
| POINT(..)  | First Street  8  |
| POINT(..)  | Second Street 32 |
| POINT(..)  | Third Street 4 |

The task: Give me the height of all houses which is store in a given triple store.

Assuming the addresses are unique identifiers in this example, the task could be solved as follows:

    SELECT ?item ?geo ?height WHERE {
        ?item ex:address "First Street 8" .
        ?item ex:height ?height .
        ?item geo:hasGeometry ?geom .
        ?item geo:asWKT ?geo .
    }

However, this approach requires one query per table row and is not user friendly.

A better approach would be to convert the column *Address* to a query variable so that the following query could be stated:

    SELECT ?item ?geo ?height WHERE {
        ?item ex:address ?address .
        ?item ex:height ?height .
        ?item geo:hasGeometry ?geom .
        ?item geo:asWKT ?geo .
    }

SPARQL 1.1 allows this behaviour by defining a VALUES statement as follows:

    SELECT ?item ?geo ?height WHERE {
        VALUES ?address { "First Street 8" "Second Street 32" "Third Street 4" }
        ?item ex:address ?address .
        ?item ex:height ?height .
        ?item geo:hasGeometry ?geom .
        ?item geo:asWKT ?geo .
    }
    
The SPARQL Unicorn allows the user to define special queryvariables which are replace by VALUES statements of connected columns of QGIS vector layers before sending the SPARQL query to the selected SPARQL endpoint.
A user defined query in this way would look like this:

    SELECT ?item ?geo ?height WHERE {
        ?item ex:address ?_address .
        ?item ex:height ?height .
        ?item geo:hasGeometry ?geom .
        ?item geo:asWKT ?geo .
    }

The underscore in the query variable *?_address* marks the variable visibly as to be supplemented by a VALUES statement as given above.

### Using GeoSPARQL or another customized SPARQL query syntax

The supported SPARQL syntax is a matter of the triple store which is queried. The SPARQL Unicorn QGIS plugin does not restrict the usage of SPARQL extensions as long as they are matching the SPARQL syntax.
If a triple store does not support e.g. a GeoSPARQL query then the SPARQL Unicorn QGIS plugin will return an error message.

## Data Enrichment

The second function of the SPARQL Unicorn Plugin is data enrichment. Here, new columns may be added to an already existing QGIS Vector layer.

### What to enrich?

The first step of a data enrichment is to know what can be enriched. 
For example:
Given a dataset of universities including their geolocation, address and name and a triple store only properties with specific attributes might be interesting for an enrichment.
In particular properties which occur sufficiently often should be of interest.
In order to find out which properties exist and how often they are represented in a SPARQL endpoint, a "whattoenrichquery" may be defined in the triple store configuration.
An example is given below:

    SELECT (COUNT(distinct ?con) AS ?countcon) (COUNT(?rel) AS ?countrel) ?rel WHERE { 
        ?con wdt:P31 %%concept%% . 
        ?con wdt:P625 ?coord . 
        ?con ?rel ?val . 
    } 
    GROUP BY ?rel 
    ORDER BY DESC(?countrel)

This query is a template query in which the variable %%concept%% may be replaced by a Wikidata concept.
The query returns every relation linked to instances of %%concept%% and its relative occurances in relation to the individuals.
The result is interpreted by the SPARQL Unicorn QGIS plugin as a list in which most occuring proeprties are shown first.

### Data enrichment process

To enrich data from a triple store to a QGIS Vector layer, each feature included in the vector layer needs to be at best uniquely identified in the respective triple store.
This has to be done by a matching relation e.g. the name of a university which is also present in Wikidata or by a URI which is already included in the QGIS vector layer.



## Adding new triple stores using configuration files

Apart from the graphical user interface new triple stores may be added to the plugin by modifying the JSON configuration files as follows:

*triplestoreconf.json*: This configuration file is delivered on installation of the SPARQL Unicorn QGIS plugin. It is not modified and serves as a backup for a possible reset option.

*triplestoreconf_personal.json*: This configuration file is created the first time the SPARQL Unicorn QGIS plugin is started. All added triple stores will be stored within there in the following format:

    {
    "name": "Research Squirrel Engineers Triplestore",
    "prefixes": {
      "geosparql": "http://www.opengis.net/ont/geosparql#",
      "owl": "http://www.w3.org/2002/07/owl#",
      "rdf": "http://www.w3.org/1999/02/22-rdf-syntax-ns#",
      "rdfs": "http://www.w3.org/2000/01/rdf-schema#",
      "spatial": "http://geovocab.org/spatial#",
      "hw": "http://hadrianswall.squirrel.link/ontology#"
    },
    "endpoint": "http://sandbox.mainzed.org/squirrels/sparql",
    "mandatoryvariables": [
      "item",
      "geo"
    ],
    "classfromlabelquery": "SELECT DISTINCT ?class { ?class rdf:type owl:Class . ?class rdfs:label ?label . FILTER(CONTAINS(?label,\"%%label%%\"))} LIMIT 100 ",
    "geoconceptquery": "",
    "whattoenrichquery": "SELECT (COUNT(distinct ?con) AS ?countcon) (COUNT(?rel) AS ?countrel) ?rel WHERE { ?con rdf:type %%concept%% . ?con geosparql:hasGeometry ?coord . ?con ?rel ?val . }  GROUP BY ?rel ORDER BY DESC(?countrel)",
    "geoconceptlimit": 500,
    "querytemplate": [
      {
        "label": "10 Random Geometries",
        "query": "SELECT ?item ?geo WHERE {\n ?item a <%%concept%%>.\n ?item geosparql:hasGeometry ?geom_obj .\n ?geom_obj geosparql:asWKT ?geo .\n } LIMIT 10"
      },
      {
        "label": "Hadrian's Wall Forts",
        "query": "SELECT ?a ?wkt_geom WHERE {\n ?item rdf:type hw:Fort .\n ?item geosparql:hasGeometry ?item_geom . ?item_geom geosparql:asWKT ?wkt_geom .\n }"
      }
    ],
    "crs": 4326,
    "staticconcepts": [
      "http://onto.squirrel.link/ontology#Watchtower",
      "http://hadrianswall.squirrel.link/ontology#Milefortlet",
    ],
    "active": true
    }
The following configuration options exist:
* *active*: Indicates that the triple store is visible in the GUI
* *classlabelquery*: A SPARQL query OR a URL which returns a set of labels for a given list of class URIs
* *classfromlabelquery*: A query which retrieves a set of classes from a given label (useful for class searches)
* *propertylabelquery*:  A SPARQL query OR a URL which returns labels for a given list of properties
* *propertyfromlabelquery*:  A SPARQL query OR a URL which returns properties for a given list of labels
* *crs*: The EPSG code of the CRS which should be used by QGIS to interpret the data received from the triple store
* *endpoint*: The address of the SPARQL endpoint of the triple store
* *geoconceptlimit*: A reasonable limit to query considering the performance of the triple store and the data included
* *geoconceptquery*: A query to retrieve concepts associated to geometrical representations inside the triple store. The results of this query or the content of staticconcepts constitutes the list of concepts which is selectable in the graphical user interface
* *name*: The name of the triple store which is display in the user interface
* *mandatoryvariables*: A list of SPARQL query variables which have to be present in the SELECT statement (usually ?item for the URI and ?geo for the geometry but sometimes also ?lat ?lon instead of ?geo)
* *querytemplate*: A list of JSON objects representing labeled queries which may be selectable in the user interface
* *staticconcepts*: A list of concepts which are loaded to the dropdown menu of available concepts
* *prefixes*: A list of prefixes which is used by the triple store. Each prefix in this list is recognized in the query interface automatically.
* *whattoenrichquery*: A query which is sent to the triple store returning attributes and their occurance frequency for the whattoenrich dialog

The configuration file allows the definition of placeholder variables (currently only %%concept%%) in template queries.
These variables are prefixed and suffixed with %% statements.
Example:

    SELECT ?item ?label ?geo WHERE {
        ?place a <%%concept%%>.
        ?place pleiades:hasLocation ?item .
        ?item geosparql:asWKT ?geo .
        ?item dcterms:title ?label .
    } LIMIT 100

This query defines the placeholder variable %%concept%% which is replaced by the currently selected concept from the dropdown menu in the user inferface when a concept is selected.
