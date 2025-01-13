---
title: "Circular Transit Maps"
date: 2019-09-21T18:57:20+02:00
author: "Jonathan Hauser"
authorAvatar: "img/ada.jpg"
tags: ["transitmaps"]
categories: ["project"]
image: "img/project_circular_transit_maps/freiburg_converted_thick.png"
draft: false
---
Transit Maps can be found in many places. By replacing the actual road geometry with a simpler geometry like arcs the result not only
becomes more aesthetically pleasing but also more readable. Nonetheless, the original road layout shouldn't be left completely unconsidered
to avoid confusion when reading the map.
<!--more-->

# Content
1. <a href="#introduction">Introduction</a>
1. <a href="#preparation">Preparation</a>
1. <a href="#discrete_frechet_distance">Discrete Frechet Distance</a>
1. <a href="#selecting_radius">Selecting the right radius</a>
1. <a href="#simplifying">Simplifying the map</a>
1. <a href="#generating_output">Generating output</a>
1. <a href="#results">Results</a>

# <a id="introduction"></a> Introduction
Transit Maps can be found in many places. They typically reassemble the actual road network. When reading a transit map the primary information that someone wants to retrieve is which station is connected to which and what is the best route from a station to another one.
To retrieve this the actual geometry of the road network is not needed and makes the real information harder to process. Replacing the actual road geometry with a simpler geometry like circles can overcome this problem. The generated map should still reassemble the original layout as it might lead to confusion otherwise. The goal, therefore, is to simplify the map while staying as close to the original as desired.
This work is based on <a href="https://sites.google.com/site/schematicmapping/Dijk-Circular.pdf">An Automated Method for Circular-Arc Metro Maps. by van Dijk et al.</a>

# <a id="preparation"></a> Preparation
The tool that generates these circular transit maps reads the original transit maps in the GeoJSON format which is quite common for transit map representation and is also used by other tools of the chair.
For this purpose the tool also allows contents to be piped in and out to be able to chain it with other tools. The GeoJSON format stores the stations as a list of nodes and the connections between them as edges between two nodes. Each edge contains a list of subpositions which are used to align the lines to the actual roads. For the circular transit maps we are only interested in the node positions so all edges are converted into straight lines. To simplify further process the straight edges are represented as arcs with the two stations as start and end node and a radius of infinity. Later the algorithm works by always combining two adjacent arcs together to form a single larger arc. To be able to calculate those possible pairs each node stores the list of arcs it is part of. After the input file is fully read all adjacent edges are added as a merge candidate into a priority queue which compares their difference to the actual geometry. Below is an example of how the prepared data would look for a very simple map. The inner nodes of each arc is a list of nodes which lie on the arc. In the first steps, all arcs are just straight lines from one node to another and therefore the inner nodes are empty. Once two arcs are merged the new arc contains the middle point of the two arcs as its inner node.

<img src="/../../img/project_circular_transit_maps/example_map.png" title="Example map"></img>

| Node | Arcs | Position |
| --- | --- | --- |
| A | {z}   | p1 |
| B | {z, y}| p2 |
| C | {y}   | p3 |

| Arc | Start | Inner nodes | End | Radius |
| --- | --- | --- | --- | --- |
| z | A | [] | B | ∞ |
| y | B | [] | C | ∞ |

# <a id="discrete_frechet_distance"></a> Discrete Fréchet Distance
To be able to determine how close an arc is to the original geometry some kind of metric is required.
For this, the Discrete Fréchet Distance is used. The Fréchet Distance is a measure of the similarity of two curves. Intuitively it can be thought of as the minimum length of a chord that spans two points that move forward independently along two separate curves.
In the case of the transit map, one of the curves consist of a fixed number of points that correspond to the locations of the stops between the start and the endpoint of the arc.
To use these positions and also to calculate the distance in polynomial time the Discrete Fréchet Distance is used. It calculates the Frechet distance with a finite set of points for each curve and doesn't take into account the space between two discrete nodes. 
As an approximation of the arc, a list of positions lying on the arc is generated and used to calculate the Discrete Fréchet Distance. 
To ensure good results with the Discrete Fréchet Distance it is important that the points lying on the arc are neither too dense nor too far away from each other. To accomplish this the points of the arcs are simplified with the Douglas-Peucker-Algorithm and densified again to ensure enough points are available even for long straight segments before they are used for the calculation of the discrete fréchet distance. With this method, the curves consist of enough points to get reasonable results with the Discrete Fréchet Distance.

# <a id="selecting_radius"></a> Selecting the right radius
To find the optimal radius of an arc a certain set of available radii are tested and the one with the lowest resulting discrete fréchet distance is selected. 
Using only a predefined set of radii also has the effect that the resulting transit map looks even a bit cleaner as only a limited set of radii is used
Selecting the possible radii is quite important as too many radii result in higher computation time while too few might result in less simplified maps as the fréchet distances get too high if the theoretical optimal radius is far away from the given available radii.
Additionally, transit maps can differ quite a lot from one another and while some contain very large and sparse lines that require large radii others might contain small curves that require smaller radii. Often even a single transit map contains large variations. A good example is the transit map of Chicago which features a small core with large arms that reach far out.
The current implementation uses $$r_i = 500 * 2^i \ \ \ \ \  0 \le i \le 7 $$as the available radii which ensures fast calculations and enough variety even for maps like the Chicago one.

# <a id="simplifying"></a> Simplifying the map
For each pair of adjacent arcs, a merge candidate is created. Each merge candidate contains the two arcs it merges and a new arc that would result when merging the two arcs. The radius of the arc is calculated by using the radius with the minimum discrete fréchet distance as described above. The discrete fréchet distance of the new arc is also used to calculate a score for the merge candidate. The score is calculated as follows $$\verb|score|(arc) = \verb|discrete_frechet_distance|(arc) - 50 * |\verb|inner_points|(arc)| $$ The second term is used to give large arcs with many inner circles a bonus as they tend to have higher fréchet distances. The merge candidates are put into a priority queue which sorts the merge candidates by score.
The algorithm then selects merge candidates from the priority queue until a given maximal fréchet distance score is exceeded. This fréchet limit defaults to 500 but can be changed as a command-line argument. It ensures that the generated maps don't differ too much from the actual geometry. Selecting a larger limit produces more simplified maps while a smaller limit stays stricter to the geometry and thus isn't able to simplify the map as much. When two arcs are merged they create new merge candidates that merge the new arc with its adjacent arcs. To handle this cheaply the arc list that is stored with each node can be used. Upon merging the merged arcs are removed from the lists of the end and start point and the middle points lists are cleared and just populated with the new arc.
The priority queue might still contain merge candidates that use one of the removed arcs though. To handle this and not require a traversal of the priority queue after every merge the merge candidates are checked for availability once they have been retrieved from the priority queue. If they are not valid anymore they are just discarded and the next merge candidate is retrieved. The check for validity is done by checking if the two arcs that should be merged are each still in the arc list of the start and endpoint.

An example how the fréchet limit affects the resulting transit maps is given below with the map of Stuttgart. 

| Stuttgart limit 500 |
| --- |
| <img src="/../../img/project_circular_transit_maps/stuttgart_converted.png" title="Stuttgart limit 500" style="max-width:100%"></img> |

| Stuttgart limit 2000 |
| --- |
| <img src="/../../img/project_circular_transit_maps/stuttgart_converted_2000.png" title="Stuttgart limit 2000" style="max-width:100%"></img> |

# <a id="generating_output"></a> Generating output
Once the fréchet limit has been reached the merging is stopped and the resulting circular transit map is outputted on stdout. First, all nodes are iterated. If they are no start or end node of an arc the positions likely have changed and need to be regenerated. This is done by checking at which position of the arc the node lies and is then selected by the list of points lying on the arc which is already used for the fréchet distance calculation. The GeoJSON format doesn't natively support arcs so the arcs are instead rasterized into points lying on the arc which are then outputted as part of the edge. LOOM, the tool that is used to generate SVGs from the maps already displays nice round looking arcs with a low amount of control points. So the current implementation uses only a single additional control point in the middle of each arc segment.

# <a id="results"></a> Results
The tool has been tested against 4 different transit maps of different sizes and complexities.
The original maps and the results with the default fréchet limit of 500 rendered with LOOM can be seen below.

| Chicago original |
| --- |
| <img src="/../../img/project_circular_transit_maps/chicago.png" title="Chicago original" style="max-width:100%"></img> |

| Chicago converted |
| --- |
| <img src="/../../img/project_circular_transit_maps/chicago_converted.png" title="Chicago converted" style="max-width:100%"></img> |

| Dallas original |
| --- |
| <img src="/../../img/project_circular_transit_maps/dallas.png" title="Dallas original" style="max-width:100%"></img> |

| Dallas converted |
| --- |
| <img src="/../../img/project_circular_transit_maps/dallas_converted.png" title="Dallas converted" style="max-width:100%"></img> |

| Freiburg original |
| --- |
| <img src="/../../img/project_circular_transit_maps/freiburg.png" title="Freiburg original" style="max-width:100%"></img> |

| Freiburg converted |
| --- |
| <img src="/../../img/project_circular_transit_maps/freiburg_converted.png" title="Freiburg converted" style="max-width:100%"></img> |

| Stuttgart original |
| --- |
| <img src="/../../img/project_circular_transit_maps/stuttgart.png" title="Stuttgart original" style="max-width:100%"></img> |

| Stuttgart converted |
| --- |
| <img src="/../../img/project_circular_transit_maps/stuttgart_converted.png" title="Stuttgart converted" style="max-width:100%"></img> |













