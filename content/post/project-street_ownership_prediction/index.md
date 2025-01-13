---
title: "Predicting Ownership of Streets in OpenStreetMap Using Machine Learning"
date: 2022-11-24T11:55:33+01:00
author: "Christoph Röhrl"
authorAvatar: "img/ada.jpg"
tags: ["OSM", "OpenStreetMap", "Streets", "Machine Learning"]
categories: ["project"]
image: "img/project_street_ownership_prediction/private_streets_heidelberg.png"
draft: false
---

In the Field of Network Infrastructure Planning, having to deploy something on
private property can have a big impact in terms of cost as well as overall efficiency.
This project investigates to which degree public available data from
OpenStreetMap about streets and their properties can be used to predict their ownership status.

<!--more-->

## Content

+ [Introduction](#introduction)
+ [Retrieving the Data](#retrieving-the-data)
+ [Preparing the Data](#preparing-the-data)
    - [Ownership Ground Truth](#ownership-ground-truth)
    - [Detecting Dead Ends](#detecting-dead-ends)
+ [Neural-Network Training](#neural-network-training)
+ [XGBoost Training](#xgboost-training)
+ [Conclusion](#conlusion)

## Introduction

In the context of fiber-optic-cable roll-out planning, knowing as much as possible about the concerned streets early on can
significantly improve the planning-process and reduce deployment costs. One of those important properties is the
ownership-status of a street. Deploying infrastructure on private property can be way more expensive than on public ground.

In Germany, a lot of streets nearby cities, especially near residential areas, are privately owned but are assigned to the public,
so telling their ownership status just by looking at them is often not possible. Most of the time, getting the official ownership data from a city's administration is not possible.

In the following, we will examine how data from OpenStreetMap together with Machine Learning can be used to predict the ownership property.

## Retrieving the Data

This project is focusing solely on the streets from the city of Heidelberg. This is not only because for this research purpose, the amount of streets is sufficient but also because it's the only city we have official data from to evaluate our findings.

The street data itself is retrieved via QLever, a SPARQL engine which can (along many other things) efficiently query the entire OpenStreetMap data.

The request for QLever looks as follows:

    PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
    PREFIX ogc: <http://www.opengis.net/rdf#>
    PREFIX osm: <https://www.openstreetmap.org/>
    PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
    PREFIX geo: <http://www.opengis.net/ont/geosparql#>
    PREFIX osmkey: <https://www.openstreetmap.org/wiki/Key:>
    PREFIX osmrel: <https://www.openstreetmap.org/relation/>
    SELECT ?osm_id ?geometry ?highway ?surface ?oneway ?length ?maxspeed ?service
        ?lanes ?sidewalk ?access ?width WHERE {
    ?osm_id osmkey:highway ?highway .
    OPTIONAL {?osm_id osmkey:surface ?surface} .
    OPTIONAL {?osm_id osmkey:oneway ?oneway} .
    OPTIONAL {?osm_id osmkey:length ?length} .
    OPTIONAL {?osm_id osmkey:maxspeed ?maxspeed} .
    OPTIONAL {?osm_id osmkey:service ?service} .
    OPTIONAL {?osm_id osmkey:lanes ?lanes} .
    OPTIONAL {?osm_id osmkey:sidewalk ?sidewalk} .
    OPTIONAL {?osm_id osmkey:access ?access} .
    OPTIONAL {?osm_id osmkey:width ?width} .
    ?osm_id rdf:type osm:way .
    ?osm_id geo:hasGeometry ?geometry .
    osmrel:285864 ogc:contains ?osm_id .
    }

As one can see, a lot of properties get requested, independent of their availability. So for example, if the maximum speed of a specific street is unknown or not tagged, the gab will be filled with a none value.

After the query is finished, the results can be downloaded as a csv file and get processed from there.

## Preparing the Data

### Ownership Ground Truth

Because the official street data from Heidelberg come in the form of a shape-file, the process of creating a dataset extended by the ground truth are made on a geometrical level.

Therefore, the query results are converted into a geojson, using the geometry attribute (which always has a valid value because it's not optional).

In the following, one can see some streets of Heidelberg, partially covered by blue areas which are the streets owned by the city.

![Figure 1: Overlapping Layers](/img/project_street_ownership_prediction/heidelberg_streets.png)

With QGis, an open-source Software which can view and edit geodata, all streets which are covered by the blue area get removed.
After some cleaning up and dealing with corner cases, all remaining streets are not State or City property and therefore privately owned.
This data can now be used to create a Dataset extended by a boolean attribute 'private_property'.

It is important to note, that in the process of cleaning up, a lot of streets from the osm dataset got removed.
This is because OSM keeps track not only of typical streets "made for cars" but also of every little footpath, sidewalk
or dirt road there is. Those types of roads/paths are not relevant in our context, which is why they were dropped.

### Detecting Dead Ends

Now we have a dataset which has a fair amount of attributes, including the ground truth.
Looking at all the privately owned streets, one can see that a lot of them are dead ends, which
is why it seems to be practical to derive an additional attribute for them using the geometry of every street.

The dead-end detection works as follows:

- Every street geometry is basically a set of ordered coordinate points, called locations.
- Connected or crossing streets have some locations which are the same.
- So if a location is only part of a single street and a start/end point at the same time,
  this specific street has a dead-end.


## Neural-Network Training

With the use of the PyTorch_Tabular framework, a basic neural network is trained using the following, optimized parameters:

    model_config = CategoryEmbeddingModelConfig(
        task="classification",
        layers= "40-40-40", # Number of nodes in each layer
        activation="LeakyReLU", # Activation between each layers
        loss="CrossEntropyLoss"
    )

90% of the total data are used for training and 10% for testing.
With the optimized parameters, the neural network achieves adequate results:

+ accuracy:  0.8069 (ratio of correctly classified data points)
+ precision: 0.8552 (ratio of positive (private) predictions which are correct)
+ recall:    0.8397 (ratio of 'detected' positive values)


## XGBoost Training

With the use of the XGBoost framework, a Gradient Boost Model is trained, which uses a method that
combines a lot of weak learners (mostly simple decision trees) into a powerful ensemble.

Using Grid Search (which finds the best values for multiple parameters) for multiple rounds, the model
gets optimized with the following settings:

    parameters = {
        'max_depth': 20,
        'learning_rate': 0.1,
        'gamma': 0,   # min_loss_split
        'reg_lambda': 5, # l2_regularization on weights
        'scale_pos_weight': 2
    }

Again, 90% of the data are used for training and 10% for testing.
With this model, we get the following results:

+ accuracy:  0.7801 (ratio of correctly classified data points)
+ precision: 0.7867 (ratio of positive (private) predictions which are correct)
+ recall:    0.8890 (ratio of 'detected' positive values)

## Conlusion

This proof of concept shows that it's possible to predict the ownership status
of streets with public available data from OpenStreetMap and Machine Learning. While the overall
accuracy of the best model still has some room for improvement, the detection of privately owned streets
works well. In a context of fiber-optic-cable roll-out planning where deployment of infrastructure on private
property must be obviated, the previous trade off between accuracy and recall can be worthwhile.

Further improvements can be made particularly in the size of the training data,
so adding a few more cities from different regions could not only boost the accuracy of the
model, but also make it better in terms of generalization.
