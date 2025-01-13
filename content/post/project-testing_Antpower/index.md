---
title: "Testing and improving Antpower on the Simbench networks"
date: 2022-06-01T12:12:46+02:00
author: "Metty Kapgen"
authorAvatar: "img/project_testing_Antpower/avatar.png"
tags: [FraunhoferISE, Antpower, Simbench]
categories: []
image: "img/project_testing_Antpower/ant.jpg"
draft: false
---

The project aims to test and improve Lukas Gebhard's Antpower on the Simbench Networks provided by Fraunhofer ISE. 
In short Lukas has developed a solver that is able to produce a cheap expansion plan given a low voltage grid. In this project we aim to test his solver on a small and synthetic set of low-voltage grids called the Simbench networks. At its core, Antpower uses the ant colony optimization algorithm
to produce cheap expansion plans. Those are needed as the current state of the grid can contain several overloaded components which need to be upgraded with better but more expensive cable types.
This project will show that, given a large computation time, Lukas' Antpower produces good solutions. 
<!--more-->


## Content

1. [Introduction](#introduction)
2. [Motivation](#motivation)
3. [Problem](#problem)
4. [Solvers](#solvers)
3. [Simbench Networks](#simbench)
4. [Test Scenarios](#test)
5. [Setup](#setup)
6. [Results](#results)
7. [Explanation of the results](#explanation)
8. [Improvements](#improvements)
9. [Conclusion](#conclusion)

## Introduction <a name="introduction"></a>

Antpower is a software developed by Lukas Gebhard during his master thesis called [“Expansion Planning of Low-Voltage Grids Using Ant Colony Optimization”](http://ad-publications.informatik.uni-freiburg.de/theses/Master_Lukas_Gebhard_2021.pdf). His code provides a noble approach at searching low cost solutions for expanding low-voltage grids, which suffer from overloaded lines and/or nodes, by using the so-called ant colony optimization algorithm. In his thesis, Lukas confirms the claims about Antpower by running it against a local search algorithm on a small grid of a German village. This project aims to assess the claims made and test them on several different [Simbench networks](#simbench) that were provided by the Fraunhofer ISE. In addition to that, the project also lays out a small improvement proposal of the current Antpower by memorizing already tested upgrades of the grid.

## Motivation <a name="motivation"></a>

Lukas’ thesis states that Antpower resulted in an 64% cheaper expansion against the best solution found by the local search algorithm. However those results were deducted from only one test scenario. Thus this project aims to strengthen the claims made in the thesis, by applying both algorithms on a much larger testset called the Simbench Networks. Simbench Networks are specially designed to contain several constraint violations in order to challenge both algorithms in finding cheap solutions.

## Problem <a name="problem"></a>

A Low-Voltage grid can be overloaded on their lines as well as on their nodes/buses. The problem one is trying to solve consists of finding the cheapest expansion plan possible while also getting rid of any constraint violations. There are two types of constraint violations the solver can differentiate, either a node/bus can suffer from voltage violations or a edge/line/cable can be overloaded.
In addition to the grid, solvers also receive a small set of cable types and their pricing as input which they can use to replace current lines to create the best expansion plan possible as the output.

## Solvers <a name="solvers"></a>

In Lukas' thesis two solvers were used and compared to each other.

Antpower is a solver based on the ant colony optimization algorithm. It proposes upgraded grids by deploying several colonies of individual ants that create their own solutions from the initial grid. Given the pricing of their solution, the pheromones of the grid are updated such that "good" upgrade options are selected by ants of the following generations.

The Local Search Solver starts off its search for the cheapest expansion plan by proposing for each of its processes (an equivalent to a colony in Antpower) a random grid consisting of already installed and upgraded lines. From now on 1-opt modifications to this network are made and proposed as solutions.

## Simbench Networks <a name="simbench"></a>

![Figure 1: The Simbench networks, their amount of lines and initial number of constraint violations](/img/project_testing_Antpower/SimbenchStats.png)
<center style="margin-top:-35px;margin-bottom:55px;">Figure 1: The Simbench networks, their amount of lines and their initial number of constraint violations.</center>

The Simbench networks that enabled us to test the Antpower were initially provided by the [Simbench Project](https://simbench.de/de/) and further processed by Fraunhofer ISE. This set of networks contains six base networks of varying size and complexity. They are separated into three rural, two suburban and one urban settings. In addition to that each setting has three load scenarios with the expected loads of the network at the current time, 10 years and 25 years in the future. The further in the future a network is placed, the more constraints are violated, as more load is placed on the lines and/or nodes. 

![Figure 2: A typical Simbench network](/img/project_testing_Antpower/Grid_on_Water.png)
<center style="margin-top:-35px;margin-bottom:55px;">Figure 2: A typical Simbench network, with constraint violations marked in red</center>

## Test Scenarios <a name="test"></a>

Antpower and the local search algorithm were tested on all the Simbench networks for all three timestamps. Initially all the lines of the individual networks were made out of the “NAYY 4x 50 SE”, the weakest cable type. There are two possible line types the algorithms can choose from in order to propose an upgrade. The line types vary in price and load capacity. 
Differently to the problem in Lukas' thesis, the test scenario for the Simbench networks does not contain any new lines that could offer an alternative pathing. The problem only offers replacement options to solve the grid constraints.

## Setup <a name="setup"></a>

To let Antpower and the local search algorithm compete on the same ground, we chose the following setup of the solvers: The ACO solver should have 14 colonies, with 10 ants per colony and should run for 50 iterations. The local search solver can test an equal amount of expansion plans by giving it 14 processes and (ants x iterations of the ACO solver) 500 iterations per process.
In Lukas' Thesis the solvers were running for 1000 iterations, however that number was reduced to 50 in this project as the local search solver already reached a local minimum for that amount.
The maximum relative voltage deviation is set as a standard to 10%. In addition to that, the penalty term for violated constraints is set to 1’000’000€. Meaning, if a solver proposes/tests a network still having 12 constraint violations, the network will cost at least 12’000’000€. If an expansion plan costs less than 1’000’000€ it does not have any constraint violations anymore. The rest of the settings were adopted from Lukas’ thesis.

## Results <a name="results"></a>

![Figure 3: Best Price Results](/img/project_testing_Antpower/stats.png)
<center style="margin-top:-35px;margin-bottom:55px;">Figure 3: Best price results (in €) from Antpower and the Local Search Solver on all the Simbench Networks</center>

The figure above shows the best price Antpower found against the best price the Local Search Algorithm proposed for each of the networks’ scenarios. As we can see, the price advantage that Antpower reaches against its adversary is much slimmer than the 64% stated by Lukas Thesis. We also notice that the local search solver does find feasible expansion plans for each network while Antpower doesn't. One can notice that some networks, for example “Rural 1” do not need any upgrades as no constraints are violated, thus replacing lines would only add costs. That is why the best found expansion price for both solvers is zero. On the other hand we also have certain networks such as “SemiUrban 5” for which the Ant Colony Solver was not able to find a feasible expansion plan, as the cheapest price it found is a multiple of the penalty term.
Figure 4 gives a look into the solver's capabilities in an extended scenario. It shows the best prices found for both solvers running for much longer by doing many more iterations on the Semiurban5 network in scenario one and two. (14 colonies, 10 ants, and 1000 iterations). 

![Figure 4: Antpower and Local Search Algorithm for 50 and 1000 iterations](/img/project_testing_Antpower/long_stats.png)
<center style="margin-top:-35px;margin-bottom:55px;">Figure 4: Solvers running on SemiUrban5 with 50 and 1000 iterations</center>

## Explanation of the results <a name="explanation"></a>

The reason for those results may be that, in contrast to the real-world network used in Lukas’ thesis, the Simbench networks are considerably less complex in their sizing and structure (see Figure 2). All the Simbench networks are so-called “spanning trees”, whereas the real world networks have a much higher complexity with many nodes and more upgrade options. Such networks also risk having circles in their structures due to new lines being added. This feature does not exist on the Simbench network.

![Figure 5: The network of Lukas thesis](/img/project_testing_Antpower/LukasThesisGrid.PNG)
<center style="margin-top:-35px;margin-bottom:55px;">Figure 5: The network tested in Lukas thesis</center>

Another explanation comes from the small number of iterations both algorithms did. In general one can say that the local search algorithm converges much quicker to possible upgrades of the network, as it starts off with a random spanning tree of upgraded and maintained lines. If the network consists of many constraint violations, this gives a clear advantage to the local search, as the ant-colony optimization algorithm tries to find a solution initially starting with the already existing grid. Comparing the best-found expansion plan prices of figure 4, one can see that by giving Antpower more iterations to find cheap solutions, it does approach the best pricing of local search. One can also note that the local search algorithm does not make any considerable progress in between running for 50 or 1000 iterations.

## Improvements <a name="improvements"></a>

This project also aimed to achieve some sort of a speedup of the actual Antpower solver. The improvement was achieved by implementing a memory layer in the solver that is able to memorize already tested networks, so that “PyPSA” (a python library called “Python for Power System Analysis’')’s powerflow does not have to run several times for the same network. Antpower tends to propose certain expansion plans several times in order to make progress. This however comes at a large cost for the overall running time, as the power flow calculation, which is responsible for testing the network for violated constraints, takes the most amount of time in each iteration of the solver. The network storage is implemented by hashing the list of all the current lines and adding it with the cost of the expansion plan into a simple dictionary. The hashing is done using sha256. This implementation did improve the average running time of a single iteration/ the average time it takes the solver to check an upgraded network.

![Figure 6: Antpower with and without improvement](/img/project_testing_Antpower/speedup.png)
<center style="margin-top:-35px;margin-bottom:55px;">Figure 6: Antpower with and without the Speedup while considering the amount of grids where powerflow could be skipped because of the proposed improvement</center>

In figure 6 one can see the time in seconds it took the ACO solver on average to propose and test one expansion plan. The speedup of this feature can be considerably (up to 95%) in the case of a small grid such as the “Rural 1” network. Many grids could be skipped because they were tested several times. However, on the other hand, it can also result in a slowdown as one can see on the “SemiUrban 5” network. This slowdown can be explained through the size of those networks. Large networks (also “Rural 3”) seem to weaken the improvement proposal, as many more expansion options can be proposed by the solvers. That is why close to no grids were tested more than one time. Thus under certain conditions, encrypting and storing those networks can serve as an overhead by exceeding the time one could gain from this.

Nevertheless, the overall speedup measured as the average of speedups on all the Simbench networks is 78,8%.

## Conclusion <a name="conclusion"></a>

Antpower is a viable option to consider in the problem setting described above. It is able to find cheap solutions given the network has a certain amount of complexity to it. 
In general the local search algorithm seems to perform better on small spanning networks with many constraint violations, while Antpower and more generally the ant-colony optimization algorithm seems to be better at finding good solutions on larger, more complex and less violated networks given it can be run for an extended amount of time. 
In future works one could add new lines as an option for the expansion plan, to get an even more realistic comparison between the Simbench and the real-world problems.
