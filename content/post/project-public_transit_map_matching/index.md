---
title: "Project Public Transit Map Matching With GraphHopper"
date: 2021-01-04T11:16:35+01:00
author: "Michael Fleig"
authorAvatar: "img/ada.jpg"
tags: ["map-matching", "GTFS", "OSM", "public transit", "schedule data"]
categories: ["project"]
image: "img/project_public_transit_map_matching/webapp.png"
draft: false
---

# Contents

1. [Introduction](#introduction)
1. [Our map matching approach](#approach)
    1. [Finding Candidates](#finding-candidates)
    1. [Path finding between candidates](#path-finding-between-candidates)
    1. [Hidden Markov Model HMM](#hidden-markov-model)
    1. [Turn restrictions](#turn-restrictions)
1. [Evaluation](#evaluation)
    1. [Metrics](#metrics)
    1. [Results Stuttgart](#results-stuttgart)
    1. [Results Victoria-Gasteiz](#results-victoria-gasteiz)
1. [Further research](#current-problems-and-further-research)


# Introduction

*TransitRouter* is a web tool for generating shapes of GTFS (*General Transit Feed Specification*) feeds for bus routes using the map matching approach described in the paper [Hidden Markov Map Matching Through Noise and Sparseness](https://www.ismll.uni-hildesheim.de/lehre/semSpatial-10s/script/6.pdf).

*TransitRouter* uses the OSM routing engine [GraphHopper]() and a modified version the [GraphHopper Map Matching library]() that enables turn restrictions and tries to prevent inter hop turns.

*GraphHopper* is a fast and memory efficient routing engine written in Java. It has built in support for different weighting strategies (e.g. fastest and shortest path routing), turn restrictions (based on OSM meta data), turn costs and most important the ability to specify custom routing profiles. As public transit has quite different traffic rules we created a specialized *bus profile*. 

The quality of the results are compared with the original GraphHopper Map Matching library (*GHMM*) and [*pfaedle*](https://github.com/ad-freiburg/pfaedle) a similar tool developed by the chair of Algorithms and Data Structures at the University of Freiburg. 

The source code of TransitRouter is available on [GitHub](https://github.com/fleigm/TransitRouter).


# Approach
Given a trip \\(T\\)  with an ordered sequence of stations \\(S = (s_0, s_1, s_2, ..., s_n)\\) we want to find its path \\(P\\) through our street network graph \\(G=(V, E)\\).
First we discuss how we can find possible candidates in G for every \\(s_i\\) and how we find a path between candidates. Then how we can find the most likely sequence of candidates and finally how we enable turn restrictions.

![text](/../../img/project_public_transit_map_matching/street_graph.svg)
Stations \\(s_0, s_1, ..., s_6\\). Red: candidates within radius \\(r\\) around \\(s_i\\). Blue: Path of the bus.

## Finding candidates
The GPS positions of our stations might not be accurate so we cannot use them directly to find our path. 

For each station \\(s_i\\) we want to find a set of possible candidates \\(C_i\\). To construct \\(C_i\\) we use the following approach:

For every edge \\(e_j \in E\\) within a radius \\(r\\) around \\(s_i\\) calculate the projection \\(p_{i,j}\\) of \\(s_i\\) on \\(e_j\\). Then for each outgoing edge \\(e_k\\) of \\(p_{i,j}\\) we add a candidate \\(c_i^{k} = (p_{i,j}, e_k)\\) to \\(C_i\\). So a candidate consists of a position in our road network and a direction in which we can drive.

Note that we could have used *orientation less* candidates consisting only of the projection \\(p_{i,j}\\) but we need the direction to enable turn restrictions which we will see at a later point.



## Path finding between candidates
As a path finding strategy either shortest or fastest routing can be used.

For feeds with short distances between stations shortest routing might produce better results that fastest routing. An example can be seen in the evaluation chapter.

We use our own public transit optimized bus profile for *TransitRouter* and *GHMM*.

## Hidden Markov Model *HMM*
To find the most likely sequence of candidates we use a Hidden Markov Model (*HMM*) with our stations \\(s_i\\) as observations and our candidates \\(C_i\\) as observations. The approach is based on the paper [Hidden Markov Map Matching Through Noise and Sparseness](https://www.ismll.uni-hildesheim.de/lehre/semSpatial-10s/script/6.pdf).

Given a optimal sequence of candidates we construct our final path \\(P\\) by combining the paths between the candidates in our sequence.

### Emission probability
The emission probability \\(p(s_i | c_i^k)\\) describes the likelihood that we observed \\(s_i\\) given that \\(c_i^k\\) is a matching candidate.

We use the great circle distance \\(d\\) between the station\\(s_i\\) and its candidate node \\(c^i_k\\) and apply a weighting function with the tuning parameter \\(\sigma\\).

$$d=\|s_i - c_i^k\|_{\text{great circle}}$$
$$p(s_i | c_i^k) = \frac{1}{\sqrt{2\pi}\sigma}e^{-0.5(\frac{d}{\sigma})^2}$$



### Transition probability
For the transition probability \\(p(c_i^k \rightarrow c_{i+1}^j)\\) describes the likelihood that our next state will be \\(c_{i+1}\\) given our current state is \\(c_i^k\\).
We use the distance difference \\(d_t\\) between the great circle distance of the two stations \\(s_i, s_{i+1}\\) and the length of the road path between the two candidates \\(c_i^k, c_{i+1}^j\\) and apply out weighting function with tuning parameter \\(\beta\\).

$$d_t = | \|s_i - s_{i+1}\|_{\text{great circle}} - \| c_i^k - c_{i-1}^j \|_{\text{route}} |$$
$$p(c_i^k \rightarrow c_{i+1}^j)=\frac{1}{\beta}e^{-\frac{d_t}{\beta}}$$

![text](/../../img/project_public_transit_map_matching/hmm.png "title here")



## Turn restrictions
For our bus routes we want to prevent forbidden and unlikely turns. GraphHopper has built in support for turn restrictions based on osm meta data. But if we calculate the path between candidates only using there position we run into the problem of inter hop turns.

Consider the following example:

![text](/../../img/project_public_transit_map_matching/inter_hop_turn_restrictions.png)

Having no information from which direction we arrived at candidate \\(c^0_1\\) the most likely path to \\(c^0_2\\) would include a full turn which is unlikely for a bus and might even be forbidden at that position by traffic rules.

In GraphHopper we can specify a start and end edge when finding the path between two nodes. We use this to enable turn restrictions between hops.
Given two candidates \\(c_1 = (u_1, e_1), c_2 = (u_2, e_2)\\). Let \\(v\\) be the neighbor of \\(u_2\\) connected by \\(e_2\\). Then we calculate the path \\(P_1\\) from \\(u_1\\) starting with edge \\(e_2\\) to \\(v\\) ending with \\(e_2\\). 
When we combine the most likely paths \\(P_i\\) to get the whole path \\(P\\) we remove the last edge from every \\(P_i\\).



# Evaluation

We evaluate *TransitRouter* and *GHMM* on the GTFS feeds of Stuttgart (S) and Victoria-Gasteiz (VG) with both fastest and shortest routing and following tuning parameters:
$$\sigma=10, \\: \beta=1.0, \\: \text{candidate search radius } r = 10$$

## Metrics

To evaluate the quality of our generated shapes we use three different metrics where we compare a generated path P with the corresponding path Q of the ground truth.

### Average Frèchet Distance \\(\delta_{a_F}\\)
We split the paths \\(P\\) and \\(Q\\) into an equal number of segments such that the segments in \\(P\\) have a length of 1m. Then \\(\delta_{a_F}\\) is the average of the Frèchet Distance between the segments in \\(P\\) and \\(Q\\).


### Percentage of unmatched hop segments \\(A_N\\)
A *hop segment* is the path between two station / hops.
A *hop segment* is mismatched its Frèchet Distance is \\(\geq\\) 20m.
$$A_N = \frac{\text{\#unmatched hop segments}}{\text{\#hop segments}}$$

### Accuracy
The accuracy is the percentage of trips that are below a given threshold of \\(A_N\\).

### Percentage of length of unmatched hop segments \\(A_L\\)
$$A_L = \frac{\text{length of unmatched segments}}{\text{length ground truth}}$$


## GraphHopper Map Matching base line
The GraphHopper Map Matching (*GHMM*) library is a one to one implementation of [Hidden Markov Map Matching Through Noise and Sparseness](https://www.ismll.uni-hildesheim.de/lehre/semSpatial-10s/script/6.pdf).

To be used as a base line we removed the filtering of close observations described in chapter 4.1 as this removes consecutive stations that are to close to one another.

The differences to TransitRouter are:
- uses orientationless candidates
- no support for turn restrictions and turn costs
- no support for inter hop turn restrictions


## Results Stuttgart

Average Frèchet Distance \\(\delta_{a_F}\\) histogram
![](/../../img/project_public_transit_map_matching/stuttgart.avg_fd.png)

Accuracy
![](/../../img/project_public_transit_map_matching/stuttgart.accuracy.png)

Percentage of unmatched hop segments \\(A_N\\)
![](/../../img/project_public_transit_map_matching/stuttgart.an.png)

Percentage of length of unmatched hop segments \\(A_L\\)
![](/../../img/project_public_transit_map_matching/stuttgart.al.png)


## Results Victoria-Gasteiz

Average Frèchet Distance \\(\delta_{a_F}\\) histogram
![](/../../img/project_public_transit_map_matching/vg.avg_fd.png)

Accuracy
![](/../../img/project_public_transit_map_matching/vg.accuracy.png)


Percentage of unmatched hop segments \\(A_N\\)
![](/../../img/project_public_transit_map_matching/vg.an.png)


Percentage of length of unmatched hop segments \\(A_L\\)
![](/../../img/project_public_transit_map_matching/vg.al.png)



## Current problems and further research

### Inter-hop turn restrictions are not properly reflected in the HMM
GraphHopper uses a time based penalty to prevent unnecessary turns but this is not reflected in our transition weight function where we only use the distance of our path. So inter hop turns are not always prevented. 

### Path finding difficulties with turn restrictions
With turn restrictions enabled for some trips GraphHopper is not able to find any path between the candidates of two stations. If this happens we disable turn restrictions for that particular trip. For Stuttgart this affects around \\(10\\%\\) of all trips. For Victoria-Gasteiz none of the trips is affected.

At the moment we were not able to find the reasons why GraphHopper is not able to find any path. This issue needs further investigation.

### Use OSM metadata
OSM provides useful information about public transit routes which might increase the quality of the generated shapes.

### Enable other vehicle types
Currently *TransitRouter* only supports bus routes. With new vehicle profiles we could add support for tram, subway and rail public transit routes.