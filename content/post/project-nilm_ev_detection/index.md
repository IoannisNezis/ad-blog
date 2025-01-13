---
title: Detection of Electric Vehicle Charging Events Using Non-Intrusive Load Monitoring
date: 2022-05-10T16:00:50+01:00
author: "Rohit Kerekoppa Ramesha"
authorAvatar: 
tags: []
categories: ["non-intrusive load monitoring","machine learning", "time series forecasting"]
image: "/img/project_nilm_ev_detection/main21.png"
draft: false
---

Non-intrusive load monitoring (NILM) is the process of using the energy consumption of a house as a time series, which is the sum of the consumptions of the individual appliances to predict the individual appliance's consumption time series. NILMTK<a href=#ref1><sup>[1]</sup></a> is an open-source toolkit for comparative analysis of NILM algorithms across various datasets. The goal of this project is to use NILMTK to predict the energy consumed while charging electric vehicles from overall energy consumption of a house using a synthetic dataset.
<!--more-->
*This project is the outcome of a cooperation between Fraunhofer ISE in Freiburg and the Algorithms & Data Structures Chair of the University of Freiburg.*

<div>
<p><b>CONTENTS</b></p>
<ol>
<li><a href="#div1">Introduction</a></li>
<li><a href="#div2">Problem Statement</a></li>
<li><a href="#div3">Advantages of NILM</a></li>
<li><a href="#div4">Motivation</a></li>
<li><a href="#div5">Overview</a></li>
<li><a href="#div6"style="text-decoration: none;">NILMTK</a></li>
<li><a href="#div7">Contributions</a></li>
<li><a href="#div8">Different Algorithms for NILM</a></li>
<li><a href="#div9">Datasets</a></li>
<li><a href="#div10">Evaluation Metrics</a></li>
<li><a href="#div11">Basic Statistics of Various Datasets</a></li>
<li><a href="#div12">Experiments</a></li>
<li><a href="#div13">Conclusion</a></li>
<li><a href="#div14">Future Works</a></li>
<li><a href="#div15">Appendix</a></li>
<li><a href="#div16">Acknowledgments</a></li>
<li><a href="#div17">References</a></li>
</ol>
</div>
<div id ="div1">
<h2>1. Introduction <br></h2>
<p>Energy monitoring is one of the most important aspects of energy management because there is a need to monitor the energy consumption of buildings before planning some of the technical measures to reduce energy consumption. Energy monitoring is important to help end users to save energy by taking energy-saving measures such as using energy efficient devices, more efficient use of electrical equipment and eliminating unwanted energy activity.  With the advent of renewable energy supply to the power grids energy management and monitoring is now more important as it is needed to help stabilize these grids. Energy monitoring not only helps end users reduce their electricity bill, but also is an important step that is needed to reduce emission of greenhouse gasses and combat climate change.  In order to reduce the burden of the power sector by its major challenges like the cost of electricity, energy crisis and global warming, some of the critical inefficiencies of the sector can be spotted and removed using load monitoring at a very low cost.</p>
<p>The two main ways of monitoring energy usage are Intrusive Load Monitoring (ILM) and Non-Intrusive Load Monitoring (NILM). Intrusive Load Monitoring (ILM) involves the installation of sensors at every appliance in order to monitor the power consumed by them. Using these sensor readings, we can monitor the energy consumption at an appliance level. Non-Intrusive Load Monitoring (NILM) is the process of deconstructing the aggregate energy consumption into its individual appliances as seen in Figure 1. This process does not require the intrusion into the individual appliances in order to monitor their power consumption.</p>
<figure>
<center><img src = "/img/project_nilm_ev_detection/NILM_concept.jpg", alt='Non Intrusive Load Monitoring Concept '>
  <figcaption>Figure 1: Non-Intrusive Load Monitoring concept<a href=#ref6><sup>[6]</sup></a></figcaption>
</center>
</figure>
<br>
<p>NILM can be formulated as either classification problem or regression problem. The regression problem is when the algorithm needs to predict the power consumption of each device at each time interval. NILM can also be used for classification by determining whether the device is ON or OFF instead of predicting its consumption at each time interval.
</div>

<div id ="div2">
<h2>2. Problem Statement </h2><br>

<p>Let the time series of aggregate measurements \(Y=(Y_1,Y_2,… ,Y_T )\)where \(Y_t\in R^+\) represent the energy or power measured in Watt-hours or Watts consumed by the building at time t. This is considered to be the aggregate of energy consumed by the individual appliances. The building facility is assumed to have m appliances and for each appliance the energy signal is represented as \(X=(X_{i1},X_{i2} ,… ,X_{iT})\) where \(x_{it} \in R^+.\)<br>
\[ Y_t=\sum_{i=1}^{m} x_{it}+\epsilon \text{, where }\epsilon \text{ represents the error at time t.}\] 
<p> The aim of NILM is to retrieve the unknown signal \(X_i\) when the aggregate signal \(Y\) is given.
</div>
<div id ="div3">
<h2>3. Advantages of NILM </h2><br>
<ul>
<li>Detailed information regarding how much energy is being used by individual appliance can be obtained with the help of NILM. This information allows the consumer to figure out which appliances are consuming a high amount of energy in their house and helps end users minimizing their electrical consumption.
</li>
<li>Since NILM can detect which machines consumes the most energy, the end users can know not to use these appliances when electricity is either costly or has a high carbon footprint.
</li>
<li>Peak demand is the highest amount of energy used during a 15-minute period during the month. This peak demand determines the rate at which the end users (who consume more than 100 MWh a year) are charged for the electricity they consume. With the help of NILM, industries can identify when they are using the most power each day, along with which machines they are using at that time. This information can help industries to find ways to reduce their peak demand.
</li>
</ul>
</div>
<div id ="div4">
<h2>4. Motivation</h2><br>

<p>In transition to a renewable energy system, many objects with high connection powers as for example electric vehicles, PV systems and heat pumps are installed to the low voltage electrical grids. With increasing number of such participant in the grid, it becomes a more complex task to keep the grid stable. In Germany these devices have to be registered in big databases<a href=#ref8><sup>[8]</sup></a> and hence are theoretically known to the grid operator but in reality, the knowledge of grid operators is often incomplete. Besides, electric vehicles are moved when used and may change the grid connection point. Energy monitoring is a necessary solution for energy management that allows acquiring appliance specific energy consumption statistics that could further be used to conceive load scheduling strategies for optimal energy usage. In this project we detect electric vehicles from synthetic data and show the suitability of the used algorithms for this task.	
<p>When the energy consumed is above 10000 kWh per year<a href=#ref7><sup>[7]</sup></a>, the smart meter data will be transferred at a 15-minute rate. Since the grid operators recive the smart meter data at a 15-minute rate, it makes sense to test the performance of NILM algorithms at a sample rate of 15 minutes. However, most of the existing research done by the scientific community in the field of NILM is done at a higher sample rate i.e., lower than 2 minutes.  It would be useful to see how well these algorithms perform at this sample rate (15 minutes) and check how the lower sample rate affects the performance of the algorithm.
</div>
<div id ="div5">
<h2>5. Overview</h2><br>
<p>In this project, first the performance of the algorithms from NILMTK at 15-minute sample rate is compared. This would help identify which algorithm performs the best at the 15-minute sample rate in a real dataset. Then the sensitivity to the sample rate is pursued for the best performing algorithm by a comparison at 15-minute sample rate and 5-minute sample rate. The sensitivity towards higher number of datapoints in training in 5-minute sample rate is analysed for the best performing algorithm by a comparison of 15-minute sample rate and 5-minute sample rate with similar number datapoints. The performance of the best performing algorithm in predicting power consumed while charging electric vehicles is analysed and is compared with other algorithms in the synthetic dataset. Performance of this algorithm in electric vehicle charging event detection is analysed and compared with a simple baseline model.
</div>
<div id ="div6">
<h2>6. NILMTK</h2><br>
<p>NILMTK is an Open-source toolkit for comparative analysis of NILM algorithms across various datasets. It also provides a pipeline from data sets to metrics to lower the entry barrier for researchers. NILMTK was created for three main reasons. Firstly, it allows the comparison of state-of-the-art approaches. Secondly, it allows the comparisons of algorithm’s performance on various datasets, so that it can be verified if the approach can be generalized to new households. Thirdly, it gives users access to a stable set of metrics that help researchers access the performance of the algorithms for various use cases. 
<br><br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/workflow.jpg", alt='NILMTK Workflow'>
  <figcaption>Figure 2: NILMTK Workflow </figcaption>
</center>
</figure>
<br>
<p>The NILMTK workflow is shown in Figure 2. The data from the Dataset is first converted to NILMTK-DF which is the standard energy disaggregation data structure used by the toolkit. Parsers for the different datasets are available which convert the dataset into NILMTK-DF. Now various different statistics can be observed in order to analyze the dataset. NILMTK provides statistical and diagnostic functions which provide a detailed understanding of each data set. Preprocessing functions are available that help mitigating challenges with NILM datasets. NILMTK provides the easy use of various different algorithms in order to disaggregate the data. Evaluation of results are possible with the Metrics that are provided by the toolkit. NILMTK now also provides an API in order to run various different experiments. The API makes running experiments extremely quick and efficient, with the emphasis on creating finely tuned reproducible experiments where model and parameter performances can be easily evaluated at a glance.
</div>
<div id ="div7">
<h2>7. Contributions</h2><br>
<ul>
<li>Creation of the synthetic dataset and adding metadata information so that the dataset is in a format that can used by NILMTK.
<li>Analysis of the different datasets by visualising the data.
<li>Adapting the NILMTK API so that one could load the saved models of different algorithms. 
<li>Adding early stopping to all the deep learning based algorithms to halt the training of these algorithms at the right time.
<li>Adding new metrics to evaluate the results.
<li>Conducting different experiments mentioned in the 'Overview' section.
</ul>
</div>
<div id = "div8">
<h2>8. Different Algorithms for NILM</h2><br>
<ol>
<h3>
<li>Simple Algorithms</li>
</h3>
<ul>
<h3>
<u><li>Hart <a href=#ref11><sup>[11]</sup></a></li></u>
</h3>
<p>The first algorithm for NILM was proposed by George Hart in 1985. The algorithm first divides the measurements along the time in sequence of time of “same power”, considered as steady state. The signal is then considered to be a sequence of stable states. A new sequence starts when the level of power change. Thus, edges in the steady time series data can be detected, which signifies that an appliance has changed its state.  In the algorithm in the toolkit, the output of hart’s algorithm (steady time series data) is assigned to the appliance categories which, maximize the algorithm’s accuracy. This is normally used as baseline model for the NILM problem.
<h3>
<u><li>Mean</li></u>
</h3>
<p>
The mean algorithm is a simple algorithm that calculates the mean power state for each appliance. It then predicts all appliances to be in ON state and returns mean power for all the appliances. It is also used as a baseline algorithm and performs comparably well in some situations.
<h3>
<u><li>Combinatorial Optimisation (CO)</li></u>
</h3>
<p>
The Combinatorial Optimization algorithm is similar to the knapsack problem. Here each appliance is assumed to be able to in a small number of states (k), each of which has a specific power consumption. The main aim of the algorithm is to select which appliances are in which state, such that the difference between the aggregate meter reading and the sum of the power consumption of appliance selected is minimized in each time step.  CO cannot handle large number of appliances as its time complexity is exponential.

</ul>
<h3>
<li>Deep Learning Based Algorithms</li></h3>
<p>The optimiser used is the Adam optimizer and the loss function used is the Mean square error in all the deep learning based algorithms. The default hyperparameter settings values are used. The Sequence length used in all the experiments is 99.
<p>
Early stopping is used while training these neural networks in order to avoid overfitting. When no improvement was seen while for 10 epochs, the training would stop and the best model would be saved. 

<ul>
<h3>
<u><li>Denoising Autoencoder (DAE)<a href=#ref10><sup>[10]</sup></a></li></u>
</h3>
<p>
An Autoencoder is an unsupervised artificial neural network that first efficiently compresses the data using an encoder and then learns to reconstruct the input from the reduced form.  A denoising autoencoder (DAE) is an autoencoder which attempts to reconstruct a clean target from a noisy input. DAE was used for NILM by creating one model per appliance that considers the Main meter reading (aggregate reading) as a noisy input. The network then reconstructs the clean power demand of the target appliance.  DAE gets main meter reading of specific length and then outputs the target appliance consumption for the same sequence length. Architecture of DAE is as follows
<br>
<ol style = " text-align: left;
 margin-left: 15%;" >
<li >Input with length optimized to the appliance</li>
<li>1D Convolution: {filters: 8, kernel size: 4, activation: linear}</li> 
<li>Fully connected: {size:input length × 8, activation: ReLU}</li>
<li>Fully connected: {size:128, activation: ReLU}</li>
<li>Fully connected: {size:input length × 8, activation: ReLU}</li>
<li>1D Convolution: {filters: 1, kernel size: 4, activation: linear}</li>

</ol>
<br>
<h3>
<u><li>Recurrent Neural Network (RNN)<a href=#ref10><sup>[10]</sup></a></li></u>
</h3>
<p>
Recurrent Neural Network (RNN) is a type of neural network that has internal memory and works very well with sequential data. This network receives a sequence of main meter readings and outputs a single value of power consumption of the target appliance. Instead of having a normal RNN which has the vanishing gradient problem, LSTMs are used. Architecture of RNN is as follows
<br>
<ol style = " text-align: left;
 margin-left: 15%;" >
<li>Input with length optimised to the appliance</li>
<li>1D Convolution: {filters: 16, kernel size: 4, activation: linear}</li>
<li>Bidirectional LSTM: {number of units: 128, activation: tanh}</li>
<li>	Bidirectional LSTM: {number of units: 256, activation: tanh}</li>
<li>	Fully connected: {size:128, activation: tanh}</li>
<li>Fully connected: {size:1, activation: linear}</li>
</ol>
<br>
<h3>
<u><li>Gated recurrent unit (GRU)<a href=#ref9><sup>[9]</sup></a></li></u>
</h3>
<p>
This network is very similar to the RNN network but replaces the LSTMs with a more lightweight RNN called Gated Recurrent Units (GRU). Similar to the DAE network,Window GRU gets main meter readings of specific length and then outputs the target appliance consumption for the same sequence length. Architecture of GRU is as follows
<br>
<ol style = " text-align: left;
 margin-left: 15%;" >
<li>Input with length optimised to the appliance</li>
<li>1D Convolution: {filters: 16, kernel size: 4, activation: linear}</li>
<li>Bidirectional GRU: {number of units: 64, activation: tanh, dropout: 0.5}
</li>
<li>Bidirectional GRU: {number of units: 128, activation: tanh, dropout: 0.5}</li>
<li>	Fully connected: {size:128, activation: tanh}</li>
<li>Fully connected: {size:1, activation: linear}</li>
</ol>
<br>
<h3>
<u><li>Sequence to Sequence (Seq2seq)<a href=#ref12><sup>[12]</sup></a></li></u>
</h3>
<p>
Sequence-to-sequence learning is about training models to convert sequences from one domain to sequences in another domain. The sequence to sequence learning model learns a regression map from the main meter sequence to the corresponding target appliance sequence. The neural network maps a sliding window Yt:t+W−1 of the aggregate input power to corresponding windows Xt:t+W−1 of the output appliance power. Since there are multiple predictions for a particular time, we take the mean for each sliding window that contains that time. Architecture of Seq2seq is as follows
<br>
<ol style = " text-align: left;
 margin-left: 15%;" >
<li>Input sequence with length W : Yt:t+W −1</li>
<li>1D Convolution: {number of filters: 30; filter size: 10} </li>
<li>1D Convolution: {number of filters: 30; filter size: 8}</li>
<li>1D Convolution: {number of filters: 40; filter size: 6}</li>
<li>1D Convolution: {number of filters: 50; filter size: 5}</li>
<li>1D Convolution: {number of filters: 50; filter size: 5}</li>
<li>Fully connected: {number of units: 1024} </li>
<li>Output: {Number of units:W }</li>
</ol>
<br>
<h3>
<u><li>Sequence to Point (Seq2point)<a href=#ref12><sup>[12]</sup></a></li></u>
</h3>
<p>
Sequence to point (Seq2point) is a similar model to Seq2seq but is trained to predict only the midpoint element of that sliding window. Thus, the output of this model only has 1 node whereas the output of Seq2Seq has more number of nodes. We expect the state of the midpoint element of that appliance should relate to the information of the aggregate power before and after that midpoint. The neural network maps a sliding window Yt:t+W−1 of the aggregate input power to the midpoint element of the corresponding window of the target appliance. This allows the neural network to focus its representational power on the midpoint of the window, rather than on the more difficult outputs on the edges, yielding more accurate predictions. Architecture of Seq2point is as follows
<br>
<ol style = " text-align: left;
 margin-left: 15%;" >
<li>Input sequence with length W : Yt:t+W −1</li>
<li>1D Convolution: {number of filters: 30; filter size: 10} </li>
<li>1D Convolution: {number of filters: 30; filter size: 8}</li>
<li>1D Convolution: {number of filters: 40; filter size: 6}</li>
<li>1D Convolution: {number of filters: 50; filter size: 5}</li>
<li>1D Convolution: {number of filters: 50; filter size: 5}</li>
<li>Fully connected: {number of units: 1024} </li>
<li>Output: {Number of units:1, activation: linear}</li>
</ol>
<br>
</ul>
</ol>
</div>
<div id ="div9">
<h2>9. Datasets</h2><br>
<p>NILM datasets can be divided into a dataset with low-frequency being up to 1 Hz and high-frequency when above that. So, the dataset that contains electricity consumption for all the appliances(sub-meters) and main meter (aggregate consumption) at a rate of at least one measurement per second is considered high-frequency datasets.  In this project we make use of three datasets, two publicly available datasets from literature as benchmark and a synthetic dataset. These datasets are described in the sequel.
<ol>
<h3>
<li>REDD<a href=#ref5><sup>[5]</sup></a></li>
</h3>
<p>
The Reference Energy Disaggregation Data Set (REDD), is a publicly available dataset containing electricity usage of 6 households for a period of about 2 months. REDD was the first public energy dataset that was released by MIT in 2011.
<p>The basic statistics and experiments on the dataset are shown in the appendix 
<h3>
<li>UK-Dale<a href=#ref4><sup>[4]</sup></a></li>
</h3>
<p>
UK-Dale is a publicly available dataset from the UK recording Domestic Appliance-Level Electricity usage for 5 households. Each household was recorded for different periods of time and the first household contains readings for approimately 4 years. 
<h3>
<li>Synthetic Dataset(Synpro)<a href=#ref3><sup>[3]</sup></a></li>
</h3>
<p>
The synthetic dataset is created using the Synpro tool which was developed at ISE. This tool allows to simulate the power demand for households based on the harmonized European time usage study HETUS. Each house in this dataset contains power time series for the entire year of 2017 at a sample rate of 15 minutes. Additionally to the demand of the household, charging of Electric Vehicles at home with different charging powers is simulated. A converter was needed that converted the output of the tool into a format used by NILMTK. This dataset consists of 15 households. 
<p>
In this dataset, 4 houses are of type “Single Family house”, 8 of type “Multi-family house” and 3 are of type “Large Multi-family house”. These houses have different number of occupants ranging from 1-8. Each house have charging stations that charged at one of the 3 charging powers: 3.7 kW, 7.2kW, 11kW.
</ol>
</div>
<div id ="div10">
<h2>10. Evaluation Metrics</h2><br>
Many different metrics were used for evaluation
<br>
<ol>
<h3>
<li>Mean Absolute Error(MAE)</li>
</h3>
<p>
This is a risk metric corresponding to the arithmetic mean of the absolute error loss. 
<p>
If ŷi is the predicted value of the i-th sample, and yi is the corresponding true value, then the mean absolute error (MAE) over n samples is defined as<br>
$$MAE(y,\hat{y})=\frac{1}{n_{sample}}\sum_{i=0}^{n_{sample}-1} |y_i-\hat{y}_i| $$
<h3>
<li>Root mean square error (RMSE) </li>
</h3>
<p>
This metric computes  the square root of the average of the set of squared differences between prediction and actual observation.
<p>
If ŷi is the predicted value of the i-th sample, and yi is the corresponding true value, then the Root mean squared error (RMSE) over n samples is defined as 
<br>
$$RMSE(y,\hat{y})=\sqrt{\frac{1}{n_{sample}}\sum_{i=0}^{n_{sample}-1} (y_i-\hat{y}_i)^2} $$ 
<p>
RMSE is considered as the more important metric than MAE since the errors are squared before they are averaged, the RMSE gives a relatively high weight to large errors.  Thus, RMSE is more desirable than MAE because punishes large errors more.
<h3>
<li>Normalised Disaggregation Error (NDE) </li>
</h3>
<p>
The comparison between appliances having a high difference in power consumption is problematic using RMSE. In order to compare the error between the different appliances this metric is used.  
<p>
If ŷi is the predicted value of the i-th sample, and yi is the corresponding true value, then the normalised disaggregation error (NDE) over n samples is defined as<br>
$$NDE(y,\hat{y})=\sqrt{\frac{\sum_{i=0}^{n_{sample}-1} (y_i-\hat{y}_i)^2}{\sum_{i=0}^{n_{sample}-1} (y_i)^2}}$$ 
<p>
NDE allows the comparison between error in prediction of energy consumed in charging vehicles in different houses in the synthetic dataset even though they have different input settings as the error is normalized.
<h3>
<li>Confusion matrix</li>
</h3>
<p>
The confusion matrix is a table that contains four entries (True Positives, False Negatives, True Negatives, False Positives). Since the models try to predict the energy consumed by an appliance and not if the appliance is on or off, the appliance is considered on if its value is above a certain threshold(T). Threshold T is set to 10% of the maximum power of the appliance, that was seen in the ground truth.
<p>
Let predicted value of energy consumed by a device by a model be P and ground truth value of energy consumed by a device be G.<br>
The prediction at a particular time point is considered to be True Positive (TP) if both P and G are above T.<br>
The prediction at a particular time point is considered to be True Negative (TN) if both P and G are below T<br>
The prediction at a particular time point is considered to be False Positive (FP) if P is above T but G is below T<br>
The prediction at a particular time point is considered to be False Negative (FN) if P is below T but G is above T
<h3>
<li>Accuracy, Precision, Recall, F1 score</li>
</h3>
<p>
The accuracy of the model is simply a ratio of correctly predicted observation to the total observations. Using the results from the confusion matrix the accuracy is given by <br>
$$Accuracy = \frac{(TP+TN)}{(TP+FP+FN+TN)} $$
<br>
<p>
The precision of the model is the ratio of correctly predicted positive observations to the total predicted positive observations. It can be formulated as <br>
$$Precision = \frac{TP}{(TP+FP)} $$
<br>
<p>
The recall of the model is the ratio of correctly predicted positive observations to the total actual positive observations. It can be formulated as<br>
$$Recall = \frac{TP}{(TP+FN)} $$
<br>
<p>
F1 score of the model is the weighted average of Precision and Recall. Therefore, this score takes both false positives and false negatives into account. The F1 score is a better metric than accuracy when there is an uneven class distribution as the accuracy of the model can be largely contributed by a large number of True Negatives (if the device is mostly off). F1 score can be formulated as<br>
$$F1 score = 2*\frac{(Precision*Recall)}{(Precision+Recall)}$$
<br><br>
</ol>
</div>
<div id ="div11">
<h2>11. Basic Statistics of various Datasets</h2><br>
<ol>
<h3>
<li>UK-Dale</li>
</h3>
<p>UK Dale contains data from 5 households and each house has different number appliances. We present a basic overview of the data from house 1.
<p>
In figure 3 the fraction of energy consumed by the 15 highest energy consuming devices over the entire 4-year period in house 1 of the UK Dale dataset is visualized as pie chart. In house 1, it can be observed that the fridge freezer, Light, washer dryer and dish washer are some of the appliances that consume the most amount of energy.<br><br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/UKDale_Figure6.jpg", alt='Figure 6 Fraction of energy consumption of top 15 appliances in house 1 of the UK Dale dataset'>
  <figcaption>Figure 3 Fraction of energy consumption of top 15 appliances in house 1 of the UK Dale dataset</figcaption>
</center>
</figure>
<br>
<p>
Figure 4 shows when the appliances are on for a period of 10 days in house 1 of UK Dale dataset. Appliances are considered on when they consume more than 10Watts at that timepoint. Fridge freezer is a device that is always on, whereas the washer dryer and dish washer are appliances that are only turned on few times a week.<br><br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/UKDale_Figure7.jpg", alt='Figure 4 On-off graph for appliances'>
  <figcaption>Figure 4 On-off graph for appliances  for a 10 day period in house 1 of UK Dale dataset </figcaption>
</center>
</figure>
<br>
<p>Figure 5 shows the appliance and Main meter consumption for a single day in house 1 of UK Dale dataset. Only the top 7 appliances are taken into account. The peaks are caused by the kettle, dish washer, kettle and toaster. These appliances cause similar peaks and use similar amount of energy while turned on. 
 <br><br>
 <figure>
<center><img src = "/img/project_nilm_ev_detection/UKDale_Figure8.jpg", alt='Figure 8 Appliance and aggregate consumption for a single day'>
  <figcaption>Figure 5 Appliance and aggregate consumption for a single day in house 1 of UK-Dale </figcaption>
</center>
</figure>
<br>
<p> UK Dale is a comprehensive dataset with a high number of recorded appliances. It also is clean and contains few missing sections. This dataset has readings for over 4 years which is beneficial for training and testing neural networks.
<h3>
<li>Synthetic Dataset (Synpro)</li>
</h3>
<p>The synthetic dataset contains 15 houses, all possessing the same appliances. An overview of the different houses is given in Table 1.
<p>
Figure 6 shows the fraction of energy consumed by all devices in house number 1, Figure 7 shows the fraction of energy consumed by all devices in house number 12. As seen below the almost 50 percent of the energy consumed by house 1 is due to charging the EV, whereas in house 12 charging EV is approximately only 20 percent of the overall energy used. This is because house 1 is of type “Single Family home” with one occupant and house 12 is of type “Multi Family Home” with 8 occupants in total.  This gives an impression of the variety of composition of energy consumption in this dataset.
<table>
    <tbody>
        <tr>
            <td width="73" valign="top">
                <p>
                    House Number
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    House type
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    Number of occupants
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    Charging rate (Kilowatt)
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    Number of electric vehicles
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    1
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Single Family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    1
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    3.7
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    2
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Single Family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    2
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    7.2
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    3
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Single Family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    3
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    11
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    4
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Single Family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    4
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    3.7
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    5
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    1
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    7.2
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    6
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    1
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    3.7
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    7
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    2
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    11
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    8
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    2
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    7.2
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    9
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    3
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    11
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    10
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    3
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    3.7
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    11
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    4
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    11
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    12
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    4
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    7.2
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    13
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Large Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    4
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    3.7
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    1
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    14
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Large Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    6
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    7.2
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    2
                </p>
            </td>
        </tr>
        <tr>
            <td width="73" valign="top">
                <p>
                    15
                </p>
            </td>
            <td width="191" valign="top">
                <p>
                    Large Multi-family house
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    8
                </p>
            </td>
            <td width="102" valign="top">
                <p>
                    11
                </p>
            </td>
            <td width="121" valign="top">
                <p>
                    3
                </p>
            </td>
        </tr>
    </tbody>
</table>
<figure>
<center><img src = "/img/project_nilm_ev_detection/Synpro_Figure9.jpg", alt='Figure 9 Fraction of energy consumption of appliance of house 1'>
  <figcaption>Figure 6 Fraction of energy consumption of appliances in house 1 of Synpro dataset </figcaption>
</center>
</figure>
<br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/Synpro_Figure10.jpg", alt='Figure 9 Fraction of energy consumption of appliance of house 12'>
  <figcaption>Figure 7 Fraction of energy consumption of appliances in house 12 of Synpro dataset </figcaption>
</center>
</figure>
<br>
<p>Figure 8 shows for a period of 1 week when the appliances are on in house 4. Appliances are considered on when they consume more than 10Watts at that timepoint. The refrigerator and freezer are always on, whereas the EV is only charged few times a week. <br><br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/Synpro_Figure11.jpg", alt='Figure 8 Plot showing when appliances are in use for house 4'>
  <figcaption>Figure 8 Plot showing when appliances are in use in house 4 of Synpro dataset </figcaption>
</center>
</figure>
<br>
<p>Synpro is a synthetic dataset which contains 15 different houses. All of these houses have different input settings and this affects the usage and energy consumption of different devices in these houses. This dataset is a comprehensive and clean dataset with no missing sections. 
<br>
</ol>
</div>

<div id ="div12">
<h2>12. Experiments</h2><br>

<ol>
<h3>
<li>Comparison between various algorithms at sample rate of 15 minutes</li>
</h3>
<br>
<p>The dataset used for a comparison of the algorithms is the UK dale dataset. This is because the REDD dataset is too small. House 1 of the dataset is used and training is done on the first 9 months of the year 2014 and testing is done on 3 months of the year 2014 for all the different algorithms. The HART algorithm is used to compare performance of the Neural network algorithms with a simple algorithm. Dish washer and washer dryer appliance are selected as the usage of these appliance is similar to that of charging EV’s as they are normally used only few times a week. As an appliance with contrary behaviour pattern, the results of Fridge are also shown in the Table 2 below. 
<table>
    <tbody>
        <tr>
            <td width="112" valign="top">
                <p>
                    Appliance
                </p>
            </td>
            <td width="116" valign="top">
                <p>
                    Algorithm
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    RMSE (Watt)
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    MAE (Watt)
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    NDE
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="6" valign="top">
                <p>
                    Fridge
                </p>
            </td>
            <td width="116" valign="top">
                <p>
                    HART
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    58.51
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    44.41
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    1.0
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    Seq2Seq
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    32.53
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    27.23
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.55
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    DAE
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    39.29
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    34.72
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.67
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    WindowGRU
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    31.67
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    25.01
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.54
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    RNN
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    29.67
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    22.90<strong></strong>
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.50
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    Seq2Point
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    <strong>29.14</strong>
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    <strong>22.20</strong>
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    <strong>0.49</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="6" valign="top">
                <p>
                    Dish Washer
                </p>
            </td>
            <td width="116" valign="top">
                <p>
                    HART
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    162.91
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    20.60
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    1.0
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    Seq2Seq
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    <strong>54.99</strong>
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    8.02
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    <strong>0.35</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    DAE
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    131.92
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    29.39
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.85
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    WindowGRU
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    74.48
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    10.24
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.48
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    RNN
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    61.02
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    7.19
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.40
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    Seq2Point
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    57.42
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    <strong>6.25</strong>
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.37
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="6" valign="top">
                <p>
                    Washer Dryer
                </p>
            </td>
            <td width="116" valign="top">
                <p>
                    HART
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    198.82
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    38.75
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    1.0
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    Seq2Seq
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    89.39
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    25.18
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.45
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    DAE
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    132.72
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    41.19
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.67
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    WindowGRU
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    112.86
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    41.53
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.57
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    RNN
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    92.99
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    23.04
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.45
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    Seq2Point
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    <strong>87.13</strong>
                </p>
            </td>
            <td width="132" valign="top">
                <p>
                    <strong>20.97</strong>
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    <strong>0.44</strong>
                </p>
            </td>
        </tr>
    </tbody>
</table>
<br>
<p>The performance of Seq2Point is better than other algorithms in almost all metrics and appliances. In Seq2Point and Seq2Seq have similar results in some cases. The results of the Dish Washer and Washer dryer appliances are considered more important than the other appliances as the main aim is to check which appliance will give the best results for predicting charging events for electric vehicles. <br><br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/Seq2point_dishwasher.png", alt='Figure 9 Result of Seq2Point on appliance dish washer'>
  <figcaption style = " text-align: left">Figure 9 Result of Seq2Point on appliance 'dish washer'</figcaption>
</center>
</figure>
<br>
<p>Figure 9 shows the ground truth of the power consumed by dishwasher and the difference between ground truth and the prediction of the Seq2point algorithm. This graph helps visualise how well the algorithm performs. In most cases the Seq2point algorithm is able to predict the power consumed by the dishwasher with small errors. However, the algorithm is seen to make mistakes sometimes with the large orange lines seen in the figure.

<h3>
<li>Comparison between Seq2point with UK dale dataset at sample rate of 15-minutes and the same dataset at sample rate of 5-minutes.</li>
</h3>
<br>
<table>
    <tbody>
        <tr>
            <td width="153" valign="top">
            </td>
            <td width="114" valign="top">
                <p>
                    Appliance
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    RMSE (Watt)
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    MAE (Watt)
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    NDE
                </p>
            </td>
        </tr>
        <tr>
            <td width="153" rowspan="3" valign="top">
                <p>
                    Seq2Point sample rate 5 minutes
                </p>
            </td>
            <td width="114" valign="top">
                <p>
                    Fridge Freezer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    <strong>27.71 </strong>
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    <strong>17.78</strong>
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    <strong>0.43</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="114" valign="top">
                <p>
                    Dish washer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    <strong>42.69</strong>
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    <strong>4.19</strong>
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    <strong>0.23</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="114" valign="top">
                <p>
                    Washer Dryer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    <strong>74.68</strong>
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    <strong>12.12</strong>
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    <strong>0.34</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="153" rowspan="3" valign="top">
                <p>
                    Seq2Point sample rate 15 minutes
                </p>
            </td>
            <td width="114" valign="top">
                <p>
                    Fridge Freezer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    29.14
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    22.20
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    0.49
                </p>
            </td>
        </tr>
        <tr>
            <td width="114" valign="top">
                <p>
                    Dish washer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    57.42
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    6.25
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    0.37
                </p>
            </td>
        </tr>
        <tr>
            <td width="114" valign="top">
                <p>
                    Washer Dryer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    87.13
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    20.97
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    0.44
                </p>
            </td>
        </tr>
    </tbody>
</table>
<br>
<p>There is improvement in the algorithm performance in all appliances. When sampled at rate of 5-minutes, the amount of training data available to the algorithm is 3 times more than when sampled at rate of 15-minutes. Most deep learning algorithms give better performance when more data is available and that can be one of the reasons which explains the improvement in the performance of the algorithm at this higher sample rate.
<br>
<h3>
<li>Comparison between Seq2point with UK dale dataset at sample rate of 15-minutes with 9 months of training data and the same dataset at sample rate of 5-minutes with 3 months training data</li>
</h3>
<br>
<p>This experiment was carried out to check if the increased performance of the Seq2point algorithm when sampled at rate of 5 minutes was because of increase in amount of training data or because of lower sample rate. Training for the dataset with 15-minute sample rate was done for the first 9 months of the year and testing on the last 3 months of the year whereas, the training for the dataset with 5-minute sample rate was done for the months of September, October and November and testing for the month of December. This way it was ensured that the number of data points in training and testing for both the experiments are similar.
<br>
<table>
    <tbody>
        <tr>
            <td width="153" valign="top">
            </td>
            <td width="114" valign="top">
                <p>
                    Appliance
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    RMSE (Watt)
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    MAE (Watt)
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    NDE
                </p>
            </td>
        </tr>
        <tr>
            <td width="153" rowspan="3" valign="top">
                <p>
                    Seq2Point sample rate 5 minutes
                </p>
            </td>
            <td width="114" valign="top">
                <p>
                    Fridge Freezer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    33.97
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    24.76
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    0.56
                </p>
            </td>
        </tr>
        <tr>
            <td width="114" valign="top">
                <p>
                    Dish washer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    <strong>44.55</strong>
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    <strong>5.38</strong>
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    <strong>0.24</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="114" valign="top">
                <p>
                    Washer Dryer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    98.86
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    <strong>16.81</strong>
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    <strong>0.43</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="153" rowspan="3" valign="top">
                <p>
                    Seq2Point sample rate 15 minutes
                </p>
            </td>
            <td width="114" valign="top">
                <p>
                    Fridge Freezer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    <strong>29.14</strong>
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    <strong>22.20</strong>
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    <strong>0.49</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="114" valign="top">
                <p>
                    Dish washer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    57.42
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    6.25
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    0.37
                </p>
            </td>
        </tr>
        <tr>
            <td width="114" valign="top">
                <p>
                    Washer Dryer
                </p>
            </td>
            <td width="112" valign="top">
                <p>
                    <strong>87.13</strong>
                </p>
            </td>
            <td width="122" valign="top">
                <p>
                    20.97
                </p>
            </td>
            <td width="118" valign="top">
                <p>
                    0.44
                </p>
            </td>
        </tr>
    </tbody>
</table>
<p>The performance in terms of RMSE of the algorithm at a sample rate 15 minutes compared to 5 minutes is better for the appliances fridge and washer dryer but worse in the appliance dish washer when both experiments are done with similar number of data points. However, the performance of the algorithm is better in all cases when the sample rate was 5 minutes (9 months training time). This shows that the increased performance in the previous experiment is mainly due to increase in training data.
<br>
<h3>
<li>Performance of Seq2Point algorithm and comparison with other algorithms on the Synpro dataset</li>
</h3>
<p>Only the performance of predicting the energy consumed by the EV for charging is shown. Since charging is only usually done few times a week, most of the values in the dataset for the appliance is 0. 
<br>
<table>
    <tbody>
        <tr>
            <td width="155" valign="top">
                <p>
                    House Number
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    RMSE (Watt)
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    MAE (Watt)
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    NDE
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    1
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    66.11
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    13.35
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    <strong>0.08</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    2
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    158.15
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    26.22
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.13
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    3
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    143.68
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    16.83
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.12
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    4
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    123.85
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    18.97
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    <strong>0.25</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    5
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    82.43
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    10.68
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.09
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    6
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    85.20
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    12.41
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.14
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    7
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    203.75
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    26.10
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.20
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    8
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    195.86
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    31.62
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.23
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    9
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    162.34
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    30.95
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.12
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    10
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    142.99
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    46.68
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.16
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    11
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    143.67
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    22.63
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.14
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    12
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    182.74
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    24.43
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.17
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    13
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    213.11
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    71.99
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.16
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    14
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    492.41
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    156.60
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.18
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    15
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    378.19
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    97.63
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.18
                </p>
            </td>
        </tr>
    </tbody>
</table>
<br>
<p>
A direct comparison between the results of various houses cannot be made using RMSE as the houses have different input settings. NDE can be used to compare results between different houses as the error value is normalised. House 4 has the highest NDE. One of the reasons why house 4 is not as good as the other houses might be because charging the electric vehicles is at a rate of 3.7kW (lowest rate) and the other appliances in the houses when combined together have similar patterns. The result of house 3 and 4 are visualised in the appendix.
<p>The comparison between various algorithms in Synpro dataset are shown below. The average value of the results for each metric for all the 15 houses is shown below. Taking average for the metric RMSE and MAE is not completely accurate as some houses which have large values affect the final results shown below more than others.
<br>
<table>
    <tbody>
        <tr>
            <td width="155" valign="top">
            </td>
            <td width="155" valign="top">
                <p>
                    RMSE (Watt)
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    MAE (Watt)
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    NDE
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    Seq2Seq
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    252.8
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    61.1
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.21
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    DAE
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    388.06
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    118
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.34
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    WindowGRU
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    238.33
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    70.4
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.20
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    RNN
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    194.53
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    40.75
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    0.17
                </p>
            </td>
        </tr>
        <tr>
            <td width="155" valign="top">
                <p>
                    Seq2Point
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    <strong>184.9</strong>
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                    <strong>40.47</strong>
                </p>
            </td>
            <td width="155" valign="top">
                <p>
                   <strong>0.15</strong>
                </p>
            </td>
        </tr>
    </tbody>
</table>
<br>
Seq2Point is still the best performing algorithm in all the metrics. The performance of the Seq2point model is slightly better than RNN model. The RMSE error of Seq2point model is lower than the corresponding RMSE error in RNN model in 11 houses out of the 15 houses in the dataset. The Seq2point algorithm outperforms all other algorithms in all 15 houses in all metrics. 


<h3>
<li>Performance of Seq2point algorithm in electric vehicle charging event detection</li>
</h3>
<br>
<p>A particular timepoint is considered to be positive in the ground truth if the electricity consumed while charging is above 10% maximum power consumed while charging appliance, else it is considered negative. Similarly, A particular timepoint is considered to be positive in the prediction if the electricity consumed while charging is above 10% maximum power consumed while charging appliance, else it is considered negative. 
<p>In order to compare the performance of the algorithm in the prediction of charging events a simple baseline algorithm was used. A particular timepoint is considered to be positive in the prediction of the baseline algorithm if the electricity consumed by the entire house (aggregate readings) was above a certain threshold. The threshold between 1000 and 10000 that maximises the f1 score for the particular house was used. The ground truth is the same as the previous case.
<br><br>
<table>
    <tbody>
        <tr>
            <td width="112" valign="top">
                <p>
                    House Number
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Algorithm
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    Precision
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    Recall
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    Accuracy
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    F1 Score
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    1
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.96
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.98
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.7%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.969
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.97
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.86
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.2%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.913
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    2
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.96
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.97
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.4%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.967
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.79
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.77
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    97.6%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.783
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    3
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.94
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.95
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.6%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.945
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.98
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.85
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.7%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.909
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    4
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.86
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.94
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.3%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.896
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.92
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.65
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    98.8%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.765
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    5
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.99
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    1
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.9%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.992
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.96
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.8
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.4%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.874
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    6
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.92
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.97
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.4%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.947
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.93
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.68
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.5%
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.863
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    7
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.90
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.97
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.6
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.931
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.88
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.81
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.5
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.863
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    8
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.91
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.96
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.6
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.933
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.88
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.59
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    98.5
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.708
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    9
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.96
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.97
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.1
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.966
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.9
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.74
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.0
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.813
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    10
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.93
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.97
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    98.8
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.951
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.75
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.66
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    93.2
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.699
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    11
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.94
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.91
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.5
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.921
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.93
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.77
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.5
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.841
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    12
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.92
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.98
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    99.5
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.945
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.84
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.72
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    98.6
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.773
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    13
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.93
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.98
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    98.3
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.952
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.75
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.75
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    89.8
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.753
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    14
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.96
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.97
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    98.6
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.966
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.96
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.88
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    97.8
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.916
                </p>
            </td>
        </tr>
        <tr>
            <td width="112" rowspan="2" valign="top">
                <p>
                    15
                </p>
            </td>
            <td width="84" valign="top">
                <p>
                    Seq2point
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.91
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.92
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    97.2
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.914
                </p>
            </td>
        </tr>
        <tr>
            <td width="84" valign="top">
                <p>
                    Baseline
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    0.9
                </p>
            </td>
            <td width="73" valign="top">
                <p>
                    0.73
                </p>
            </td>
            <td width="78" valign="top">
                <p>
                    97.8
                </p>
            </td>
            <td width="72" valign="top">
                <p>
                    0.805
                </p>
            </td>
        </tr>
    </tbody>
</table>
<p>The accuracy of the model in predicting charging events is very high. This is because the task of predicting if the car is charging is a simpler than predicting how much energy is consumed while charging the car. As the car is normally only charged few times a week, most events are negative((Non charging events). While charging a car there is a significant increase in electricity consumption making it easy to predict charging events. F1 score is a better indicator compared accuracy as the dataset is imbalanced with high number of true negatives. Inclusion of another appliance which also consumes similar amount of electricity is expected to make the prediction task harder.
<p>The F1 score of the Seq2point algorithm is higher than the baseline model in all houses. The Seq2point algorithm and baseline model both have similar precision in many houses but the baseline model has a lower recall value in all houses.  This is because of the high number of false negatives. Thus, the baseline algorithm is unable to detect events when the charging power for the EV is not high.
</ol>
</div>
<div id ="div13">
<h2>13. Conclusion</h2><br>
<p>
Non-intrusive load monitoring (NILM) is used in order to estimate electrical consumption of individual appliances using the aggregate power meter reading. Different algorithms were compared in a real dataset in order to identify the best performing model. Performance of Seq2Point is better than other algorithms in the experiment. Performance of model improved when the sampling rate was 5 minutes as the amount of training data available to the algorithm is more than when the sampling rate was 15minutes. A synthetic dataset was used to simulate the power demand for charging EV’s. Performance of Seq2Point was analysed in order to predict both charging events as well power consumed while charging EV’s.  The model was able to predict with high accuracy if the EV’s were charging or not. In the synthetic dataset the Seq2point algorithm’s normalised error was higher when the pattern of power consumed while charging EV’s and other appliances were similar.
</div>
<div id ="div14">
<h2>14. Future Work</h2><br>
<p>Including the sub meter (appliance) Heat pump to the synthetic dataset. Heat Pumps consume a lot of energy.  After the addition of Heat Pumps to the dataset, energy needed to charge EV’s will no longer end up being the only appliance that consumes a lot of energy. Thus, it is expected to be harder for the model to predict accurately the charging events.
<p>Creation of new algorithm that improves the prediction of power consumed while charging EV’s. 
<p>Testing out different hyperparameter settings and how they affect the algorithm’s accuracy.
<p>Testing the accuracy of the model in a real dataset that contains energy consumed while charging EV’s
</div>


<div id ="div15">
<h2>15. Appendix</h2><br>

<ol>
<h3>
<li>REDD</li>
</h3>
<p>REDD has data from 6 households and each house has different number appliances. A basic overview of the data from house 1 shown.
Figure 10 shows the fraction of energy consumed by the 10 highest energy consuming devices over the 2-month period. The highest energy consuming appliance in the period was the fridge. The dataset also had information about energy consumed by the lights and sockets in different rooms.<br><br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/REDD_Figure3.jpg", alt='Figure 3 Fraction of energy consumption of top 10 appliances'>
  <figcaption>Figure 10 Fraction of energy consumption of top 10 appliances in  house 1 of REDD </figcaption>
</center>
</figure>
<br>
<p>Figure 4 shows for a period of 1 week when the appliances are on. Appliances are considered on when they consume more than 10Watts at that timepoint. There are some missing values in the Site meter (aggregate) values. This household has a split-phase mains supply and thus has 2 site meters. <br><br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/REDD_Figure4.png", alt='Figure 4 On-off graph for appliances'>
  <figcaption>Figure 11 On-off graph for appliances in house 1 of REDD </figcaption>
</center>
</figure>
<br>
<p>Figure 5 visualizes the appliance and aggregate consumption for a single day. This plot only shows consumptions of the top 5 highest consuming devices for that day. Peaks can be observed when the washer dryer and the dish washer are used.<br><br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/REDD_Figure5.jpg", alt='Figure 12  Appliance and aggregate consumption for a single day in house 1 of REDD'>
  <figcaption>Figure 12 the appliance and Main meter consumption for a single day REDD </figcaption>
</center>
</figure>
<br>
<p>The REDD dataset is a small dataset which only has 2 months data and has missing sections where the values for the appliances and aggregate readings are missing. To overcome this deficit, only the good sections are considered while carrying out experiments.  An aggregate of both the site meter values were used as the main meter values (aggregate value) while carrying out experiments with REDD.
<br>
<br>
<h3>
<li>Comparison between various algorithms at sample rate of 15 minutes in REDD dataset.</li>
</h3>
<p>The performance of the algorithms on the appliances dish washer and washer dryer are shown in the table. Training is done with data from house 2, house 3 and house 4 and testing is done on data from house 1. All the houses have large missing sections and these sections are not considered for training and testing. The amount of training and testing data points after using 4 different houses and ignoring the missing sections is lot lower than the experiments done with the UK Dale dataset.  
<br>

<table>
    <tbody>
        <tr>
            <td width="109" valign="top">
                <p>
                    Appliance
                </p>
            </td>
            <td width="116" valign="top">
                <p>
                    Algorithm
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    RMSE (Watt)
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    MAE (Watt)
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    F1 score
                </p>
            </td>
        </tr>
        <tr>
            <td width="109" rowspan="5" valign="top">
                <p>
                    Dish Washer
                </p>
            </td>
            <td width="116" valign="top">
                <p>
                    Seq2Seq
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    107.27
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    21.87
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.59
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    DAE
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    134.27<strong></strong>
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    31.67
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.2
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    WindowGRU
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    118.46
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    23.03
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.45
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    RNN
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    114.58
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    30.10
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.57
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    Seq2Point
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    <strong>100.40</strong>
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    <strong>21.77</strong>
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    <strong>0.64</strong>
                </p>
            </td>
        </tr>
        <tr>
            <td width="109" rowspan="5" valign="top">
                <p>
                    Washer Dryer
                </p>
            </td>
            <td width="116" valign="top">
                <p>
                    Seq2Seq
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    154.77
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    39.29
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.64
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    DAE
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    210.07
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    48.46
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.49
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    WindowGRU
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    <strong>111.83</strong>
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    <strong>27.34</strong>
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.54
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    RNN
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    194.58
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    36.83
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    0.59
                </p>
            </td>
        </tr>
        <tr>
            <td width="116" valign="top">
                <p>
                    Seq2Point
                </p>
            </td>
            <td width="137" valign="top">
                <p>
                    132.45
                </p>
            </td>
            <td width="134" valign="top">
                <p>
                    32.85
                </p>
            </td>
            <td width="123" valign="top">
                <p>
                    <strong>0.70</strong>
                </p>
            </td>
        </tr>
    </tbody>
</table>
<p>
Seq2point algorithm performs better on the appliance ‘dish washer’ but the WindowGRU algorithm performs better on the appliance ‘washer dryer’. Since the REDD dataset is not a clean dataset and has smaller amount of data the results on the UK Dale dataset must take precedence. 
<br>
<br>
<h3>
<li>Effect of sequence length hyperparameter on Seq2Point model for the Synpro dataset</li>
</h3>
<p>
The sequence length chosen for all above experiments was the default value of 99. Experiments were carried out by increasing the sequence length by 8 from 51 till 147 for each of the 15 houses. Results of how the RMSE error varies with sequence length for the first 4 houses which were of type Single Family homes can be seen in Figure 20. Figure 21 and Figure22 visualises how the RMSE error varies with the sequence length for the next 8 houses (Multi-family house) and last 3 houses (Large Multi-family house) respectively.
<br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/Sequence_length_Figure20.png", alt='Figure 13 Sequence length modifications for houses 1-4'>
  <figcaption>Figure 13 Sequence length vs RMSE for houses 1-4 of Synpro dataset</figcaption>
</figure>
<br>
</center>
<figure>
<center><img src = "/img/project_nilm_ev_detection/Sequence_length_Figure21.png", alt='Figure 13 Sequence length modifications for houses 5-12'>
  <figcaption>Figure 14 Sequence length vs RMSE for houses 5-12 of Synpro Dataset</figcaption>
</center>
</figure>
<figure>
<center><img src = "/img/project_nilm_ev_detection/Sequence_length_Figure22.png", alt='Figure 13 Sequence length modifications for houses 13-15'>
  <figcaption>Figure 15 Sequence length vs RMSE for houses 13-15 of Synpro Dataset</figcaption>
</center>
</figure>
<p>In some houses lower sequence length than the default value (99) results in lower RMSE error results whereas in other houses higher sequence length than 99 result in improved performance. A direct correlation between sequence length modifications and RMSE error cannot be made and thus the default values were used in the experiments.
<br>
<br>
<h3>
<li>Visualising the result of Seq2point algorithm on the Synpro dataset</li>
</h3><p>
Figure 16 and Figure 17 shows the ground truth of the power consumed by charging EV’s and the difference between ground truth and the prediction of the Seq2point algorithm for houses 3 and 4 of the Synpro dataset respectively.
<br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/Synpro_house3.png", alt='Figure 16  Result of Seq2Point on house 3 of Synpro'>
  <figcaption style = " text-align: left">Figure 16  Result of Seq2Point on house 3 of Synpro</figcaption>
</center>
</figure>
<br>
<figure>
<center><img src = "/img/project_nilm_ev_detection/Synpro_house4.png", alt='Figure 17  Result of Seq2Point on house 4 of Synpro'>
  <figcaption style = " text-align: left">Figure 17  Result of Seq2Point on house 4 of Synpro</figcaption>
</center>
</figure>
<p>

It can be seen that there are large errors in the prediction in house 4 in many cases and that the algorithm performs better in house 3. This demonstrates that one cannot compare performance between houses using the metric RMSE as the RMSE error in house 3 is higher than house 4. This shows that the NDE metric is a better metric to compare performance between houses.
</ol>
</div>

<div id ="div16">
<h2>16. Acknowledgments</h2><br>
<p>Special thanks to my supervisor at Fraunhofer ISE Dr. Benedikt Köpfer for helping me in this project. I would also like to thank head of the Chair Algorithms and Data Structures, Hannah Bast, and my supervisor, Matthias Hertel for this opportunity to do this wonderful project. 
</div>

<div id ="div17">
<h2>17. References</h2><br>

<ol>
<li id="ref1">Nipun Batra, Jack Kelly, Oliver Parson, Haimonti Dutta, William Knottenbelt, Alex Rogers, Amarjeet Singh, Mani Srivastava. NILMTK: An Open Source Toolkit for Non-intrusive Load Monitoring. In: 5th International Conference on Future Energy Systems (ACM e-Energy), Cambridge, UK. 2014. DOI:
<a href="http://dx.doi.org/10.1145/2602044.2602051" rel="nofollow">10.1145/2602044.2602051</a>
. arXiv:
<a href="http://arxiv.org/abs/1404.3878" rel="nofollow">1404.3878</a>
</li>
<li id="ref2">Nipun Batra, Rithwik Kukunuri, Ayush Pandey, Raktim Malakar, Rajat Kumar, Odysseas Krystalakos, Mingjun Zhong, Paulo Meira, and Oliver Parson. 2019. Towards reproducible state-of-the-art energy disaggregation. In Proceedings of the 6th ACM International Conference on Systems for Energy-Efficient Buildings, Cities, and Transportation (BuildSys '19). Association for Computing Machinery, New York, NY, USA, 193–202. DOI:
<a href="https://doi.org/10.1145/3360322.3360844" rel="nofollow">10.1145/3360322.3360844</a>
</li>
<li id="ref3">D. Fischer, A. Härtl, B. Wille-Haussmann. Model for Electric Load Profiles With High Time Resolution for German Households, in: Energy and Buildings, 2015, Vol. 92., Pages 170–179. <a href="https://doi.org/10.1016/j.enbuild.2015.01.058" rel="nofollow">https://doi.org/10.1016/j.enbuild.2015.01.058</a>
</li>
<li id="ref4">Kelly, J., Knottenbelt, W. The UK-DALE dataset, domestic appliance-level electricity demand and whole-house demand from five UK homes. Sci Data 2, 150007 (2015). <a href=https://doi.org/10.1038/sdata.2015.7>https://doi.org/10.1038/sdata.2015.7</a>
</li>
<li id="ref5">Kolter, J & Johnson, Matthew. (2011). REDD: A Public Data Set for Energy Disaggregation Research. Artif. Intell.. 25. 
</li>
<li id="ref6">Pujić, Dea & Jelić, Marko & Tomasevic, Nikola & Batic, Marko. (2020). Chapter 10 Case Study from the Energy Domain. <a href= https://doi.org/10.1007/978-3-030-53199-7_10>10.1007/978-3-030-53199-7_10.</a>
</li>
<li id="ref7">Verbraucherzentrale (2022, April 25)<a href=https://www.verbraucherzentrale.de/wissen/energie/preise-tarife-anbieterwechsel/smart-meter-die-neuen-stromzaehler-kommen-13275#:~:text=Ein%20intelligentes%20Messsystem%20%E2%80%93%20auch%20Smart,speichert%20und%20verarbeitet%20die%20Daten>https://www.verbraucherzentrale.de</a>.
</li>
<li id="ref8">Marktstammdatenregister(2022, April 25)<a href=https://www.marktstammdatenregister.de/MaStR>https://www.marktstammdatenregister.de/MaStR</a>
</li>
<li id="ref9">
Odysseas Krystalakos, Christoforos Nalmpantis, and Dimitris Vrakas. Sliding window approach for online energy disaggregation using artificial neural networks. In Proceedings of the 10th Hellenic Conference on Artificial Intelligence, 2018.
</li>
<li id="ref10">
Jack Kelly and William Knottenbelt. Neural NILM: Deep Neural Networks Applied to Energy Disaggregation. In Proceedings of the 2nd ACM International Conference on Embedded Systems for Energy-Efficient Built Environments, BuildSys ’15, pages 55–64, New York, NY, USA, 2015. ACM.event-place: Seoul, South Korea.
</li>
<li id="ref11">
G. W. Hart. Nonintrusive appliance load monitoring. Proceedings of the IEEE, 80(12):1870–1891, December 1992.
</li>
<li id ="ref12">
  Chaoyun Zhang, Mingjun Zhong, Zongzuo Wang, Nigel Goddard, and Charles
Sutton. 2018. Sequence-to-Point Learning With Neural Networks for NonIntrusive Load Monitoring. In Thirty-Second AAAI Conference on Artificial Intelligence <a href=https://arxiv.org/abs/1612.09106>https://arxiv.org/abs/1612.09106</a>
</li>

</ol>
</div>
