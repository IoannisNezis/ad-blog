---
title: "River Maps"
date: 2020-02-15T18:57:20+02:00
author: "Jianlan Shao"
authorAvatar: "img/ada.jpg"
tags: ["OSM", "LOOM", "River Maps"]
categories: ["project"]
image: "img/project_river_maps/transitmap.png"
draft: false
---
In this project, we extract the waterway system from OpenStreetMap data and render it into a way which makes it possible to track the river tributaries. We use the tool LOOM to display the relationship between waterways.

<!--more-->

# Content
- <a href="#introduction">Introduction</a>
- <a href="#codestructure">Code structure</a>
	- <a href="#container">Container</a>
	- <a href="#methods">Methods</a>	
- <a href="#possibleproblems">Possible Problems</a>
- <a href="#futureworks">Furture Works</a>

# <a id="introduction"></a>Introduction

This project extracts the information of the rivers from [OSM](https://www.openstreetmap.org/#map=12/49.4851/8.4594) data and interprets the river system into a more graphic way. More precisely, the tributaries per edge segment are regarded as lines. In the meantime, each river segment is labeled with all the tributaries so far, for example, beginning at Mannheim, the Neckar should be part of the segment that makes up the Rhine. In the end, each river will look like a single subway line starting at the source of the river, and the Rhine, for example, will look like dozens of small subway lines next to each other. We use our tool [LOOM](http://loom.cs.uni-freiburg.de/#stuttgart), which renders line graphs in a metro-map style, to present the result in the above described way. 

Here is an example illustrating this project. We have a screenshot of a map, which is an area in Hamburg. We can observe the river Elbe and other small rivers in the map, but unfortunately it is not recognizable which river consists of which tributaries. 


<table style="text-align:center; margin: 0px;">
  <tbody style="display:table;">
    <tr>
      <td><img src="/img/project_river_maps/HamburgOriginal.jpg" style="width:500px;height:346px;text-align:center;margin: 0px;"/></td>
    </tr>
    <tr>
      <td>Hamburg (original)</td>
    </tr>
  </tbody>
</table>

Now we have the result of our project expressed by LOOM of the same area. As we can see, only the information of the river system is showed on the picture, which is more straight forward to the observers. Besides, the rivers and their tributaries are represented clearly. 

<table style="text-align:center; margin: 0px;">
  <tbody style="display:table;">
    <tr>
      <td><img src="/img/project_river_maps/HamburgRendered.jpg" style="width:500px;height:350px;text-align:center;margin: 0px;"/></td>
    </tr>
    <tr>
      <td>Hamburg (rendered)</td>
    </tr>
  </tbody>
</table>

# <a id="codestructure"></a>Code structure

## <a id="container"></a>Container

One of the major catches of this project is to store the river information extracted from OSM data properly. To fulfill this task, a class named `RiverGraph` with three private map attributes is created. Here is a table interpreting these attributes.

<table style="width:100%">
<tbody style="width:100%;display:table;">
  <tr>
    <th colspan="5" style="background-color:rgba(0,0,0,.075);text-align:center;vertical-align:middle;"><h6>Attributes of the class  <code>RiverGraph</code></h6></th>
  </tr>
  <tr>
    <th style='text-align:center;vertical-align:middle'>Name</th>
    <th style='text-align:center;vertical-align:middle'>Meaning</th> 
    <th style='text-align:center;vertical-align:middle'>Key</th> 
    <th style='text-align:center;vertical-align:middle'>Type of Key</th>
    <th style='text-align:center;vertical-align:middle'>Value</th>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_nMap</td>
    <td style='text-align:center;vertical-align:middle'>Node map</td> 
    <td style='text-align:center;vertical-align:middle'>ID of nodes</td>
    <td style='text-align:center;vertical-align:middle'><code>uint64_t</code></td>
    <td style='text-align:center;vertical-align:middle'><code>RiverNode</code> instance</td>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_eMap</td>
    <td style='text-align:center;vertical-align:middle'>Edge map</td> 
    <td style='text-align:center;vertical-align:middle'>ID of edges</td>
    <td style='text-align:center;vertical-align:middle'><code>uint64_t</code></td>
    <td style='text-align:center;vertical-align:middle'><code>RiverEdge</code> instance</td>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_rMap</td>
    <td style='text-align:center;vertical-align:middle'>River map</td> 
    <td style='text-align:center;vertical-align:middle'>name of rivers</td>
    <td style='text-align:center;vertical-align:middle'><code>string</code></td>
    <td style='text-align:center;vertical-align:middle'><code>RiverName</code> instance</td>
  </tr>
</tbody>
</table>

As shown in the table, each of the map attributes has a class as it's value, that is: `RiverNode`, `RiverEdge` and `RiverName`. These classes store information for the corresponding node, edge or river. Now let's see which information is preserved in each of the classes. First of all, the class `RiverNode`.

<table style="width:100%">
<tbody style="width:100%;display:table;">
  <tr>
    <th colspan="3" style="background-color:rgba(0,0,0,.075);text-align:center;vertical-align:middle"><h6>Attributes of the class <code>RiverNode</code></h6></th>
  </tr>
  <tr>
    <th style='text-align:center;vertical-align:middle'>Name</th>
    <th style='text-align:center;vertical-align:middle'>Meaning</th> 
    <th style='text-align:center;vertical-align:middle'>Type</th>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_lon</td>
    <td style='text-align:center;vertical-align:middle'>longitude</td> 
    <td style='text-align:center;vertical-align:middle'><code>double</code></td>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_lat</td>
    <td style='text-align:center;vertical-align:middle'>latitude</td> 
    <td style='text-align:center;vertical-align:middle'><code>double</code></td>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_eInN</td>
    <td style='text-align:center;vertical-align:middle'>edge in node</td> 
    <td style='text-align:center;vertical-align:middle'><code>unordered_set<uint64_t></code></td>
  </tr>
</tbody>
</table>

Except for the longitude and latitude information of each node, the third attribute "edge in node" means the edge(s) using this node as a point in it. If the node is an intersection, there will be two or more objects in this set. This attribute will be especially useful when we come to distinguish which rivers are the upstream ones for the current river, but we will come back to this later in the methods part. Next, let's find out which features are in the class `RiverEdge`.

<table style="width:100%">
<tbody style="width:100%;display:table;">
  <tr>
    <th colspan="3" style="background-color:rgba(0,0,0,.075);text-align:center;vertical-align:middle"><h6>Attributes of the class <code>RiverEdge</code></h6></th>
  </tr>
  <tr>
    <th style='text-align:center;vertical-align:middle'>Name</th>
    <th style='text-align:center;vertical-align:middle'>Meaning</th> 
    <th style='text-align:center;vertical-align:middle'>Type</th>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_eName</td>
    <td style='text-align:center;vertical-align:middle'>edge name</td> 
    <td style='text-align:center;vertical-align:middle'><code>string</code></td>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_eType</td>
    <td style='text-align:center;vertical-align:middle'>edge type</td> 
    <td style='text-align:center;vertical-align:middle'><code>string</code></td>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_nInE</td>
    <td style='text-align:center;vertical-align:middle'>node in edge</td> 
    <td style='text-align:center;vertical-align:middle'><code>deque<uint64_t></code></td>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_rId</td>
    <td style='text-align:center;vertical-align:middle'>river id</td> 
    <td style='text-align:center;vertical-align:middle'><code>uint64_t</code></td>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_rInE</td>
    <td style='text-align:center;vertical-align:middle'>river in edge</td> 
    <td style='text-align:center;vertical-align:middle'><code>unordered_set<uint64_t></code></td>
  </tr>
</tbody>
</table>

First, we have the name and the type features stored as a string, and then we have "node in edge" which are the points describing the river line in the flow direction, this information comes directly from OSM data. Edges with the same name and intersect with each other share the same "river id", which is different from the key id of the `RiverEdge`, in which each edge has, of course, a unique key. This is a little bit tricky because in the practice we usually find out many rivers, especially the small ones having the same name. For example, in Baden-Württemberg, a state of Germany, there are 228 river edges named "Schwarzenbach", and they spread everywhere in the black forest. In this case, we have to check if the edges associated with each other, directly or indirectly, to identify if they belong to the same river. We will discuss this further in the [methods) section. "River in edge" is the upstream rivers of the current edge, it stores the river ids, rather than key ids of `RiverEdge`. The last class we want to clarify is the `RiverName`.

<table style="width:100%">
<tbody style="width:100%;display:table;">
  <tr>
    <th colspan="3" style="background-color:rgba(0,0,0,.075);text-align:center;vertical-align:middle"><h6>Attributes of the class <code>RiverName</code></h6></th>
  </tr>
  <tr>
    <th style='text-align:center;vertical-align:middle'>Name</th>
    <th style='text-align:center;vertical-align:middle'>Meaning</th> 
    <th style='text-align:center;vertical-align:middle'>Type</th>
  </tr>
  <tr>
    <td style='text-align:center;vertical-align:middle'>_eInR</td>
    <td style='text-align:center;vertical-align:middle'>edge in river</td> 
    <td style='text-align:center;vertical-align:middle'><code>unordered_set<uint64_t></code></td>
  </tr>
</tbody>
</table>

Here we have "edge in river" which contains the edge id of the rivers with the same name. Be aware, the edges here might not intersect with each other directly, or even appears in the two opposite corners of the map. That means, they can just be two irrelevant rivers sharing the same name. 



## <a id="methods"></a>Methods

We have three major jobs to process the data. The first one is to extract the needed information from OSM and to store it in the right place, which is operated by the class `OsmFilter`. Secondly, we need to reconstruct the data so that we can know the upstream and downstream relationship among the rivers, this is processed by the class `DirectedGraph`. The last one is to output the data in the proper `json` form, which is done by the class `GeoJSON`.

### <a id="dataextraction"></a>1. Data Extraction, class `OsmFilter`

As we all know, the OSM data has three kinds of elements: nodes, ways, and relations. In the case of getting river map information, only the first two kinds are relevant. In this project, the OSM data are filtered two times in the data extracting procedure. In the first round, the qualified waterways, that is: *rivers*, *streams*, *canals*, *drains*, *ditches* and *brooks* are distinguished and stored into the map `RiverEdge`, together with their names and the nodes describing the waterways. The id of these nodes is also stored into the map `RiverNode` as keys. In the second round, the geographical information (longitude and latitude) of the nodes in the map `RiverNode` is stored.

### <a id="graphconstruction"></a>2. Graph Construction, class `DirectedGraph`

To know which are the exact upstream rivers of different waterways, we try to build a directed graph from the extracted data via the following operations:

* **Add "in" and "out" information**

  As said in the container part, in the map `RiverNode`, we have a set for each node to store the edge(s) using this node. If there are multiple edges in it, this node is an intersection point. In this part, for all the beginning points, ending points, as well as intersection points, we store the number of edges going in or out. For example, at the merging point of two rivers into one, the point has two rivers coming in and one going out. The source points of all rivers have of course only one river going out and the points where the river meets the sea has only one coming in.

* **Cut edges**

  At each intersection point, that is, at the point where two or more rivers meet, if there is an edge going through this point, the edge will be cut into two. In this way, the river map becomes a properly directed graph so that the parent nodes (upstream rivers) for respective paragraphs of the rivers can be displayed more clearly.

* **Concatenation**

  If there is only one river coming and one leaving at a particular intersection point, and these two rivers have the same name, the two edges will be concatenated into one. We do this because in practice we found some rivers expressed as many successive pieces in the OSM data, and this makes the result rather untidy. Here are two screenshots representing the results before and after the concatenation.
  <table style="text-align:center; margin-top: 0px;width:70%; margin-left:15%; margin-right:15%;">
    <tbody style="display:table;">
      <tr>
        <td><img src="/img/project_river_maps/BeforeConcatenation.jpg" style="width:500px;height:288px;text-align:center;margin: 0px;"/></td>
      </tr>
      <tr>
        <td>Before concatenating</td>
      </tr>
    </tbody>
  </table>
  <table style="text-align:center; margin: 0px;width:70%; margin-left:15%; margin-right:15%;">
    <tbody style="display:table;">
      <tr>
        <td><img src="/img/project_river_maps/AfterConcatenation.jpg" style="width:500px;height:244px;text-align:center;margin: 0px;"/></td>
      </tr>
      <tr>
        <td>After concatenating</td>
      </tr>
    </tbody>
  </table>
  
  One thing to mention here is that the breaking point in the second image on the ditch "Affengraben" comes from the OSM data, that is, the starting point and the ending point of the two river edges are not identified, although they should be the same according to the usage of the OSM. We will talk more about this in the section of [possible problems](#possibleproblems).

* **Add river names**

  Here comes the last step, which is to add the river names of the upstream rivers to the downstream ones. We first build a list of the nodes without incoming edges, that is, the sources of all the rivers. Then we traverse the graph with a BFS-like approach. We go in the downstream direction. Each time we add a name to a downstream river, we deduct the outgoing river number of the current node by one, and the incoming river number of the next node by one. The name will of course only be added when it does not exist in the set of upstream river names yet. We also add the next node to the list. When the node has no edge going out, we remove it from the list. At the time when all the intersection nodes have no outgoing river, that is, when the list is empty, the name adding process is complete. 

### <a id="output"></a>3. Output, class `GeoJSON`

The output of this project is a `json` file, and the nodes in the map `RiverNode` and the edges in the map `RiverEdge` will be printed here one by one.

# <a id="possibleproblems"></a>Possible Problems

## <a id="riversdrainingtolakes"></a>1. Rivers draining to lakes

If a river drains into a lake, rather than into a sea, its route can be interrupted because there is no proper waterway between this river and the downstream river that it contributes to.  

<table style="text-align:center; margin: 0px;">
  <tbody style="display:table;">
    <tr>
      <td><img src="/img/project_river_maps/Ammersee.jpg" style="width:500px;height:526px;text-align:center;margin: 0px;"/></td>
    </tr>
    <tr>
      <td>Rivers draining to lakes</td>
    </tr>
  </tbody>
</table>
  
The above image shows the waterways around Ammersee, which is a lake in Bayern. The lake is roughly represented by the blue oval. We can see that the three streams "Fahrmannsbach", "Mühlbach" and "Kittenbach" in purple are cut off by the border of the lake. They contribute to the river Amper, but the gap caused by the lake makes it unable to observe this in the final river map.
  
## <a id="errorsfromosmdata"></a>2. Errors from OSM data

* **Repeated edges**

  Some edges in the OSM data are described repeatedly, and this will cause the problem, for example in the concatenation part.
  <table style="text-align:center; margin: 0px;width:70%; margin-left:15%; margin-right:15%;">
    <tbody style="display:table;">
      <tr>
        <td><img src="/img/project_river_maps/DuplicatedEdges.jpg" style="width:500px;height:321;text-align:center;margin: 0px;"/></td>
      </tr>
      <tr>
        <td>Repeated edges</td>
      </tr>
    </tbody>
  </table>
  
  Meet the ditch "Nördlicher Bahngraben" in Hamburg, it is marked in green color in the above picture. This ditch is represented by multiple edges, among them, four are expressed the same for two times (most of these are at the bottom right corner of the image). In this case, the intersection of those edges will be regarded as a meeting point of three or four edges, therefore it can't be concatenated properly.


* **Wrong direction of rivers**

  <table style="text-align:center; margin: 0px; width:70%; margin-left:15%; margin-right:15%;">
    <tbody style="display:table;">
      <tr>
        <td><img src="/img/project_river_maps/WrongDirection.jpg" style="width:500px;height:380;margin: 0px;"/></td>
      </tr>
      <tr>
        <td>Wrong direction of rivers</td>
      </tr>
    </tbody>
  </table>
  
  Here we have the canal "Landscheide" described in two parts, but these two edges have the same origin, which is the intersection point of them. Edges like this can also not be concatenated in the right way. In nature, it is hardly possible that the two rivers have the same source point. But in the OSM data, one should not be surprised when this happens. 

* **Breaking points**

  As shown in the methods part, breaking points lead to inaccuracy in the concatenation. Besides, breaking points also cause problems in adding river names. If two river edges are not properly linked at the same node, the upstream river names could not be passed to the next node and the river name information will be lost here. Unfortunately, breaking points of waterways is rather a common error in the OSM data, especially in small rivers. 

# <a id="futureworks"></a>Future works

## <a id="addlengthfiltertotherivers"></a>1. Add length filter to the rivers

When the map becomes large, it is quite easy that a large river has hundreds of upstream rivers. For example, in the state of Hamburg, which is a rather small state of Germany, there are more than 500 upstream rivers in the Elbe. In this case, it is almost impossible to use LOOM to render the resulting map. A map with a maximum 20 parallel river lines is more practical for LOOM. Under this condition, a filtering mechanism can be quite useful to render large scale maps, and also make the resulting map simple and elegant. 

Since the filtering mechanism influences the name adding process, the best approach to me is to add a length summing function right before it. Like the name adding function, the filter should check the rivers in the downstream direction and avoid removing rivers staying in the middle, since this will sabotage the consecutiveness of the rivers. Also, a boolean flag should be added into the values of the `RiverEdge` map, indicating if or not the corresponding edge is kept by the filter. 

## <a id="addcolorstotheedges"></a>2. Add colors to the edges

It will be nice if every upstream river has a unique color so that one can observe it better from the source to the sea. The color feature of edges, as well as nodes, is already there in the output, but currently, they are all set to <font color="#0000ff">0000ff</font>, which is blue. To update this, another attribute about color should be added to the `RiverEdge` map.

