---
title: "Energy Price Forecasting"
date: 2022-01-31T17:00:50+01:00
author: "Sneha Senthil"
authorAvatar: "/img/project-energy-price-forecasting/snake.jpg"
tags: ["machine learning", "time series forecasting", "MLP", "Residual MLP", "LSTM", "Predicting Energy Prices"]
categories: []
image: "/img/project-energy-price-forecasting/main.jpg"
draft: false
---
The project aims to predict energy prices for the next 24 hours, given the history of past features such as load, generation, prices and weather data. This data is downloaded from 2 data sources: ENTSOE and Copernicus. MLP's, residual networks and LSTMs are trained with different hyperparameters, different subsets of features and different histories and the results are compared. The residual MLP has the best results. This is likely to due to the fact that the residual MLP can adapt to changing prices better than the other models.

<!--more-->

*This project is the outcome of a cooperation between Fraunhofer IIS/EAS in Dresden and the Algorithms & Data Structures Chair of the University of Freiburg.*

<div>
<p><b>CONTENTS</b></p>
<ol>
<li><a href="#div1">Introduction</a>
<li><a href="#div2">Problem Statement</a>
<li><a href="#div3">Workflow</a>
<li><a href="#div4">Data</a>
<li><a href="#div5">Data Preprocessing</a>
<li><a href="#div6">Models</a>
<li><a href="#div7">Evaluation Metrics</a>
<li><a href="#div8">Results</a>
<li><a href="#div9">Additional Models</a>
<li><a href="#div10">Problems and Future Improvements</a>
<li><a href="#div11">Conclusion</a>
<li><a href="#div12">Appendix</a>
</ol>
</div>
<ol>
<div id ="div1">
<b><li>INTRODUCTION </b><br>
<p>There is an increasing share of renewable energies. Weather-dependent fluctuations in the amount of energy generated arise, which are directly reflected in the traded price at electricity stock exchanges. Hence, predictions of the energy price in connection with flexible electricity tariffs can help to shift peak loads to times of high availability of renewable electric energy and thus reduce carbon dioxide emissions. We attempt to predict the prices via time series forecasting.
<p>Time series forecasting is the method of making predictions into the future based on previous data. There are different types of forecasting:
<ul>
<li>Univariate forecasting: The model predicts a dependant variable based on one independent variable/feature</li> 
<li>Multivariate forecasting: The model predicts a dependent variable based on more than one independent variable/features.</li> 
<li>One-step forecast: The model predicts a single value for the next time step.</li> 
<li>Multi-step forecast: The model predicts a few time steps ahead.</li> 
</ul>
	<p>Since the model needs to predict 24 time-steps in the future and is trained on more than one feature , it is a multivariate, multi-step model. <br>
The price of energy can be dependent on multiple factors. This project considers weather data including radiation, air temperature, precipitation and wind speed as well as load and generation values.
</li>
<p>
</div>
<div id = "div2">
<b><li>PROBLEM STATEMENT </b><br>
A precise definition of the problem we intend to solve would be as follows: Given the last n-hours of data (which includes weather data, load, generation and prices), the model should be able to predict the next 24 hours of energy prices. The goal is to try out different models and identify which has the best results.
</li>
</div>
<div id = "div3">
<p>
<b><li> WORKFLOW </b><br>
<img src = "/img/project-energy-price-forecasting/workflow.png", alt='Workflow'>
<br>Each step is explained further.
</li>
</div>
<p>
<div id = "div4">
<b><li> DATA </b><br>
There are 2 publicly available datasets which contain relevant information. In both datasets, data is available at an hourly frequency. The data for Spain is downloaded, joint and preprocessed before being used for model training. <br>
Datasets:
<ol>
<li> Copernicus: Copernicus is the European Union’s Earth Observation programme. It offers information services that draw from satellite observations and in-situ data. The European Commission manages the programme. The information is free and openly accessible to users. The dataset ‘Climate and energy indicators for Europe from 1979 to present derived from reanalysis’ is used. Data is downloaded at a country level, i.e there is a separate column for each country in the EU with the respective values. The following climate variables are downloaded: wind speed, surface downwelling shortwave radiation, pressure at sea level, air temperature and total precipitation. Additionally, solar photovoltaic power generation and wind power generation are also downloaded. Each feature is downloaded as a separate .csv file. Data is available from 1979. </li>
<li> Entsoe:  This is the European Network of Transmission System Operators. Load, day ahead prices and generation can be directly downloaded for a specific country using API requests. Power generation values are provided for all sources, however since we are mainly concerned with solar and wind, we aggregate the remaining values into a single column as ‘Other energy sources’. Data is available from 2014 to 2021 and data specifically for Spain is downloaded.</li>
</ol>
<table>
<tr>
<th>Feature</th>
<th>Description</th>
</tr>
<tr>
<td> Load </td>
<td> Data about power consumption in MW </td>
</tr>
<tr>
<td> Generation </td>
<td> Energy production for solar and wind energy sources. Other energy sources are aggregated together (MW) </td>
</tr>
<tr>
<td> Prices </td>
<td> For every market time unit the day-ahead prices in each bidding zone (Currency/MWh) </td>
</tr>
</table> <p>
</div>
<div id = "div5">
<b><li> DATA PREPROCESSING </b><br>
<P>Data Preprocessing includes cleaning and correcting the data. After downloading the data, there are 2 datasets, one each from Entsoe and Copernicus. <br>
<p>Firstly, with the Entsoe data, empty rows are removed. By default, data is downloaded with the datetimes as an index column. The dates column is shifted from the index to its own column. This is done as it helps in a later step of combining the data with the Copernicus data.  Additionally, the dates and times in the Entsoe dataset are in the timezone of Madrid. This does not match with the Copernicus data which is in UTC timezone. This date column is converted to datetime datatype and converted to UTC timezone to match the Copernicus data. <br>

<p>Next with the Copernicus data: data specifically only for Spain is extracted from each file and combined together. Duplicates are dropped and any blank values are replaced with 0’s.The ‘Date’ column is converted to datetime datatype. Finally the Entsoe and copernicus datasets are merged on the ‘Date’ column <br>
<p>Following are the data columns in the dataset:
<ul><li>Date</li></ul>
From Entsoe:
<ul>
<li>Prices</li>
<li>Load</li>
<li>Solar power</li> 
<li>Wind offshore</li> 
<li>Wind onshore</li>
<li>Other Energy Sources (sum of all the other energy sources such as hydro, Nuclear, Fossil Oil, etc.)</li></ul>
From Copernicus:
<ul>
<li>Shortwave Radiation</li>
<li>Solar Pholtaic Voltage (Solar power)</li>
<li>Wind Power Onshore Generation</li>
<li>Air Temperature</li>
<li>Precipitation</li>
<li>Wind Speed</li>
</ul> <br>
<p>Relevant features are chosen. Wind offshore is not considered since all its values are 0. The power generation variables of Entsoe are chosen over Copernicus after graphical analysis of the data. A huge rise in solar power over recent years in the Entsoe data is comparable to the increase in usage of solar power over the recent years and is therefore chosen. ‘Other Energy Sources’ is also discarded as the analysis is more focused on solar and wind power. <br>
<p>Additionally the data is transformed/scaled. The whole data is split into three sets: training, validation and test set. Training set is the first 70% of the whole dataset. The rest is divided equally into the validation and training dataset. The model was evaluated on the validation set as training occurred. The test dataset was used for final evaluation after training was complete. <br>

<p>Each of the training, validation and test sets is further split into sets of features and labels.The features will have 'n' number of rows, each row containing electric and weather data for a certain hour. The number of rows in the features is determined by the number of hours we go back in the history. These features are used to predict 24 future hourly energy price values, which would be the labels. For example, if we take a history of 72 hours, each data sample would have 72 rows of features (each row containing electric and weather data for the hour) corresponding to 24 labels which are the future price values.
</div>
<div id = "div6">
<b><li>MODELS</b>
<ul><li><u>Linear Model</u>: The model used a single dense layer as the output layer, with number of output neurons = 24 (number of outputs). This is used as a baseline to compare with the results of more complex models.
</li>
<li><u> "Day Before" Baseline Models:</u> Another baseline model that can be considered is one that returns the price values of the previous 24 hours. So there is no learning and the prediction is essentially a repetition of the past values.
<li><u>MLP (Multilayer Perceptron)</u>: It is a feed-forward artificial neural network. Neural networks are inspired by biological neural networks in the brain and similarly consist of nodes that transmit information. They attempt to recognize underlying relationships in a set of data. MLPs have an input layer, one or more hidden layers and an output layer. The nodes in each layer implement a set of equations that result in an output. This predicted output is compared to the actual output to produce the error. The model attempts to reduce this error as much as possible. <br>
<p>For the MLP, the input needs to be flattened. The input before flattening would be of the form (number of samples x number of hours back x number of features). So for example, if we choose to look back 72 hours in the past for training, the data would be of the form (72x8) since each row of data has 8 features. This is flattened to a simple vector with size 576, obtained by multiplying both values. The input layer would now be receiving the data in the form (number_of_samples x 576). <br>
In this project, multiple configurations of MLPs were tried. Different histories as well as different permutations of features were tried. Following is one of the architectures used.
<p>The model is as shown below:<br><br>
<center><img src = "/img/project-energy-price-forecasting/mlp_model.png", alt='MLP MODEL'></center>
Following are the hyperparameters: <br>
<table>
<tr>
<td> number_steps_in  </td>
<td> 72</td>
</tr>
<tr>
<td> number_steps_out  </td>
<td> 24</td>
</tr>
<tr>
<td> Dense_1 neurons   </td>
<td>256</td>
</tr>
<tr>
<td> Dense_2 neurons  </td>
<td> 24</td>
</tr>
<tr>
<td> optimizer  </td>
<td>Adam</td>
</tr>
<tr>
<td> Initial learning rate   </td>
<td> 1e-4</td>
</tr>
<tr>
<td> loss   </td>
<td>Mean squared error</td>
</tr>
<tr>
<td> Epochs  </td>
<td> 300</td>
</tr>
</table> <br>
 Trainable parameters: 153,880<br>
 Regularization methods:
<ul>
<li>Early stopping: This is used to avoid overfitting. Training automatically
stops when the validation loss does not show any improvement after
10 epochs. Naturally, early stopping helps in reducing training time if
there is no improvement after a number of epochs. </li>
<li>Reduce learning rate on plateau: Reduces learning rate when validation
loss has plateaued over 10 epochs. This was particularly useful as it
was observed in earlier experiments while training a model without this
method, validation loss would plateau and not decrease at all. There is an
observation of a decrease in training and validation loss in most cases,
when the learning rate is reduced during training.
</li>
</ul>
<center><img src = "/img/project-energy-price-forecasting/loss_lr.jpg" alt="loss and learning rate"> 
<small><i> Loss when ‘reduce learning rate on plateau' is applied</i> </small>
<img src = "/img/project-energy-price-forecasting/loss_nolr.jpg" alt="loss and learning rate"> 
<small><i> Loss for the same model without reducing the learning rate on plateau</i> </small>
</center> <br>
</li>
<li><u> Residual MLP:</u>
 <p>A custom mean layer is introduced to help make
the process of training the model easier. The objective of the layer is that it
calculates hourly averages of energy prices from the past data. So its output is
24 values, each signifying the mean of energy prices over the past few days at
that specific hour. For example, given the data of the past 72 hours. This is
energy price data over three days. The mean of the values over each hour is
calculated. The final result is 24 values, a mean energy price value for each hour
of the day.
<p>The model is then forced to learn the difference between these past energy
prices and the energy prices 24 hours in the future. The differences predicted are
added to the past hourly averages and this is the final output of the model. The
idea is that training might be easier if the model only has to learn the slight
differences between energy prices, instead of predicting the energy price itself.
Input1 is all the features in the past 72 hours and input2 is the energy prices in
the past 72 hours.
<p><p>As mentioned previously, the inputs need to be flattened to a simple vector. A
similar model as the previous shown MLP is used to show the improvement in
performance.
The model is as shown below:
<center><img src = "/img/project-energy-price-forecasting/residual.png" alt = "Residual MLP Model"></center>
<p>Some details about hyperparameters:
<table>
<tr>
<td> number_steps_in  </td>
<td> 72</td>
</tr>
<tr>
<td> number_steps_out  </td>
<td> 24</td>
</tr>
<tr>
<td> Dense_1 neurons   </td>
<td>256</td>
</tr>
<tr>
<td> Dense_2 neurons  </td>
<td> 24</td>
</tr>
<tr>
<td> optimizer  </td>
<td>Adam</td>
</tr>
<tr>
<td> Initial learning rate   </td>
<td> 1e-4</td>
</tr>
<tr>
<td> loss   </td>
<td>Mean squared error</td>
</tr>
<tr>
<td> Epochs  </td>
<td> 300</td>
</tr>
</table><br>
Regularization: Early Stopping, Reduce Learning on Plateau<br>
Trainable parameters: 153,880
</li>
<li><u>LSTM:</u> LSTM is a special type of Recurrent Neural Networks. LSTM is an
improvement over RNN, because it can pass relevant information down a long
chain of sequences to make predictions. This is due to the presence of gates.
The model uses two LSTM layers, followed by a dense layer.
It was mentioned previously that the input to an MLP needs to be flattened to a
simple vector. However, this is not the case with LSTMs. The input can be
passed in the form (number of hours back x number of features). LSTMs can
easily process time series data. The input data would be of the form (number of
samples x number of hours back x number of features).
<p>The hyperparameters are as follows:
<table>
<tr>
<td> number_steps_in  </td>
<td> 72</td>
</tr>
<tr>
<td> number_steps_out  </td>
<td> 24</td>
</tr>
<tr>
<td> Lstm_1 nodes  </td>
<td>50</td>
</tr>
<tr>
<td> Lstm_2 nodes   </td>
<td> 50</td>
</tr>
<tr>
<tr>
<td>dense_1</td>
<td>24</td>
</tr>
<tr>
<td> optimizer  </td>
<td>Adam</td>
</tr>
<tr>
<td> Initial learning rate   </td>
<td> 1e-4</td>
</tr>
<tr>
<td> loss   </td>
<td>Mean squared error</td>
</tr>
<tr>
<tr>
<td>Batch size</td>
<td>64</td>
</tr>
<tr>
<td> Epochs  </td>
<td> 300</td>
</tr>
</table><br>
Regularization: Early Stopping, Reduce learning rate on plateau<br>
Trainable parameters: 33,224
</li></ul><br>
</div>
<div id = "div7">
<b><li>  EVALUATION METRICS </b>
<p>Evaluation was done on the test dataset which includes data from July 2020 till June 2021.
Evaluation is done using the following metrics:
<ul>
<li>Mean Absolute Error (MAE): Average magnitude of the errors in a set of
predictions. It’s the average over the test sample of the absolute differences
between prediction and actual observation. </li>
<li>Root Mean Square Error (RMSE): RMSE is a quadratic scoring rule that also
measures the average magnitude of the error. It’s the square root of the average
of squared differences between prediction and actual observation.</li>
</ul> </li><br>
</div>
<div id = "div8">
<b><li>RESULTS</b>
<p>The accuracy of the different models shown above was measured using the above mentioned
metrics. These values are calculated using the test dataset:
<center><table>
<tr>
<th>MODEL</th>
<th>MAE</th>
<th>MRSE</th>
</tr>
<tr>
<td>Linear</td>
<td>8.93</td>
<td>9.98</td>
</tr>
<tr>
<td>Day Before Baseline Model</td>
<td>8.19</td>
<td>9.76</td>
</tr>
<td>MLP</td>
<td>6.06</td>
<td>7.03</td>
</tr>
<td>Residual MLP</td>
<td>5.66</td>
<td>6.72</td>
</tr>
<td>LSTM</td>
<td>11.33</td>
<td>12.96</td>
</tr>
</table> </center><br>
<p><u>Graphs for model loss:</u>
<p>The following graphs show how the training and validation losses progressed during training for each model.
<table>
   <tr> 
   <td><center>Linear Model<img src="/img/project-energy-price-forecasting/linear_loss.jpg" alt="Linear Model Loss" style="width:100%"> 
    </td>
<td> <center>MLP<img src="/img/project-energy-price-forecasting/mlp_loss.jpg" alt="MLP Model Loss" style="width:100%"> </td>
</tr>
<tr>
<td><center>Residual MLP<img src="/img/project-energy-price-forecasting/res_loss.jpg" alt="Residual MLP Loss" style="width:100%"> 
    </td>
<td> <center>LSTM<img src="/img/project-energy-price-forecasting/lstm_loss.jpg" alt="LSTM Loss" style="width:100%"> </td>
</tr>
</table> <br>
<p>As observed, the MLP and the Residual MLP do a good job of reducing the training
loss as well as the validation loss. The validation loss is much closer to the training loss in both
instances, so there is some confidence that overfitting is avoided.
<p><u>Graphs for evaluation metrics </u>
<table>
   <tr> 
   <td><center>Linear Model<img src="/img/project-energy-price-forecasting/linear_metrics.jpg" alt="Linear Model Metrics" style="width:100%"> 
    </td>
<td> <center>MLP<img src="/img/project-energy-price-forecasting/mlp_metrics.jpg" alt="MLP Model Metrics" style="width:100%"> </td>
</tr>
<tr>
<td><center>Residual MLP<img src="/img/project-energy-price-forecasting/res_metrics.jpg" alt="Residual MLP Metrics" style="width:100%"> 
    </td>
<td> <center>LSTM<img src="/img/project-energy-price-forecasting/lstm_metrics.jpg" alt="LSTM Model metrics" style="width:100%"> </td>
</tr>
</table>
<p> In these graphs, one can observe the Mean Average error and the Root Mean Square error for the test dataset for the different models. These values are observed along the Y-axis. 
<p> It is observed that after 2021-01, there are spikes in the error values. One possibility as to why this occurs could be due to the Corona crisis. This may have led to sudden changes in energy prices that have not been seen before. The idea is that the Residual MLP should be able to overcome this as it is only learning the change in energy prices from the previous few days, instead of the actual energy price. This could be helping as the average MAE and average RMSE have decreased for the Residual MLP.
<p><u>Graphs for price prediction</u><br>
The following plots show the predictions of the different models for four randomly selected days from the test data. <br> 
<b>Case 1 </b>

<table>
   <tr> 
   <td><center>Linear Model<img src="/img/project-energy-price-forecasting/1_baseline.jpg" alt="Baseline 1" style="width:100%"> 
    </td>
<tr>
<td> <center>MLP<img src="/img/project-energy-price-forecasting/1_mlp.jpg" alt="MLP 1" style="width:100%"> </td>
</tr>
<tr><td><center>Residual MLP<img src="/img/project-energy-price-forecasting/1_res.jpg" alt="Residual MLP 1" style="width:100%"> 
    </td></tr>
<tr><td> <center>LSTM<img src="/img/project-energy-price-forecasting/1_lstm.jpg" alt="LSTM 1" style="width:100%"> </td>
</tr>
</table><br>
<b>Case 2 </b>

<table>
   <tr> 
   <td><center>Linear Model<img src="/img/project-energy-price-forecasting/2_baseline.jpg" alt="Baseline 2" style="width:100%"> 
    </td>
<tr>
<td> <center>MLP<img src="/img/project-energy-price-forecasting/2_mlp.jpg" alt="MLP 2" style="width:100%"> </td>
</tr>
<tr><td><center>Residual MLP<img src="/img/project-energy-price-forecasting/2_res.jpg" alt="Residual MLP 2" style="width:100%"> 
    </td></tr>
<tr><td> <center>LSTM<img src="/img/project-energy-price-forecasting/2_lstm.jpg" alt="LSTM 21" style="width:100%"> </td>
</tr>
</table><br>
<b>Case 3 </b>

<table>
   <tr> 
   <td><center>Linear Model<img src="/img/project-energy-price-forecasting/3_baseline.jpg" alt="Baseline 3" style="width:100%"> 
    </td>
<tr>
<td> <center>MLP<img src="/img/project-energy-price-forecasting/3_mlp.jpg" alt="MLP 3" style="width:100%"> </td>
</tr>
<tr><td><center>Residual MLP<img src="/img/project-energy-price-forecasting/3_res.jpg" alt="Residual MLP 3" style="width:100%"> 
    </td></tr>
<tr><td> <center>LSTM<img src="/img/project-energy-price-forecasting/3_lstm.jpg" alt="LSTM 3" style="width:100%"> </td>
</tr>
</table><br>
<b>Case 4 </b>

<table>
   <tr> 
   <td><center>Linear Model<img src="/img/project-energy-price-forecasting/4_baseline.jpg" alt="Baseline 4" style="width:100%"> 
    </td>
<tr>
<td> <center>MLP<img src="/img/project-energy-price-forecasting/4_mlp.jpg" alt="MLP 4" style="width:100%"> </td>
</tr>
<tr><td><center>Residual MLP<img src="/img/project-energy-price-forecasting/4_res.jpg" alt="Residual MLP 4" style="width:100%"> 
    </td></tr>
<tr><td> <center>LSTM<img src="/img/project-energy-price-forecasting/4_lstm.jpg" alt="LSTM 4" style="width:100%"> </td>
</tr>
</table><br>
<p>As observed the Residual  MLP seems to do the best job at predicting energy price values. This is
especially noticed with the 4th graph set where the other models’ predictions are
nowhere close to the actual values. 
<p>One observation is that the prices in the test dataset are quite high and have values that have never been seen before in the training dataset. Therefore, the models struggle to accurately predict a higher energy value when there has not been a similar dramatic change in the values of the other features. This is where the Residual MLP can tend to do better, since it is created to predict the change in the hourly energy prices and not the actual prices itself. So even if the prices in the dataset are high values that the model has not been trained on, it can accurately predict the change in the prices and therefore can make a more accurate price prediction.
<p>The MLP has scope for improvement with the
addition of another dense layer. Even the results of a Residual MLP with an additional dense layer are observed. The results of these models are mentioned in the appendix.<br>
</li>
</div>
<div id = "div9">
<b><li>ADDITIONAL MODELS</b>
<p>In order to understand the importance of all the features in the data, different
models, with different subsets of the data are trained. The model used is the Residual MLP.
<table>
<tr>
<th>FEATURES</th>
<th>MAE</th>
<th>RMSE</th>
</tr>
<tr>
<td>Prices </td>
<td>7.5</td>
<td>8.83</td>
</tr>
</tr>
<tr>
<td>Prices, load, Solar power, Wind
onshore power (Electric Data)
 </td>
<td>6.57</td>
<td>7.68</td>
</tr>
</tr>
<tr>
<td>Shortwave Radiation, Air
Temperature, Precipitation, Wind
Speed
</td>
<td>8.48 </td>
<td>9.72</td>
</tr>
</table>
<p> It can be observed from this table that prices and the electric data are more important than the weather data to accurately forecast the prices. This does not mean that the weather data is not required, but that without the electric data, price prediction would be impossible. Additionally, the prices alone are not enough for forecasting. There is an effect of other factors on the prices that are required to be considered. <br>
<p>Additionally, one feature was removed from the dataset and then the model was trained
on it. This was done for all the features one at a time. Following are the results from this
experiment:
<table>
<tr>
<th>FEATURE REMOVED</th>
<th>MAE</th>
<th>RMSE</th>
</tr>
<tr>
<td>Wind Speed </td>
<td>5.72</td>
<td>6.69</td>
</tr>
</tr>
<tr>
<td>Air Temperature
 </td>
<td>5.51 </td>
<td>6.47</td>
</tr>
</tr>
<tr>
<td>Precipitation
</td>
<td>5.77</td>
<td>6.75</td>
</tr>
<tr>
<td>Shortwave radiation
</td>
<td>7.08</td>
<td>8.19</td>
</tr>
<tr>
<td>Wind onshore power generation
</td>
<td>5.72</td>
<td>6.76</td>
</tr>
<tr>
<td>Solar power generation
</td>
<td>7.63</td>
<td>8.75
</td>
</tr>
<tr>
<td>Load 
</td>
<td>9.17</td>
<td>10.48
</td>
</tr>
<tr>
<td>Prices
</td>
<td>7.31</td>
<td>8.52
</td>
</tr>
</table>
<p>The model without ‘Air Temperature’ has a better MAE value by only 0.1. There are no
significant improvements in the other models. This is at least some evidence that these
features are impacting prices in some way and are required by the model for training.
<p> This also shows that 'Load' is the most important feature for price prediction. This is observed by the fact that without load, the MAE and RMSE are quite high. After load, the other important features observed are radiation, solar power generation and prices. Precipitation, wind power generation and wind speed improve the model a bit.
</li>
</div>
<div id = "div10">
<b><li>PROBLEMS AND FUTURE IMPROVEMENTS</b>
<p>The results could be better. It is possible that we are not taking into account all the factors that do affect the
energy prices.<br>
Some possible improvements are:
<ul>
<li>Consider seasonality of the dataset. Some features experience regular changes over the year. Taking this into consideration can help the model.
<li> Weather forecasts for the future day could also be considered to help in the price prediction for that day.
<li>Update the model to show an estimate of the prediction uncertainty to allow for
reasonable risk assessment. This would be further explored in the thesis. </li>
<li>Trying out more complex models: LSTNet, a model which combines CNN,
recurrent layer and fully connected layer, seems promising. A more complex
model might be required to learn the relationship between 8 features. This could
also be further explored in the thesis </li>
</ul>
</div>
<div id = "div11">
<b><li>CONCLUSION</b>
<p>In this experiment, I tackle the problem of forecasting future price values given past historic
hourly data. Simple MLP models (with the addition of a custom layer) are used for the
prediction. Different hyperparameter settings are explored to find the best possible setting. It is
useful to know these before trying out more complex machine learning models for price
prediction. The graph visualizations help to understand the difference in performances of the
models. It was shown how the addition of the mean layer helped to improve the MAE without
adding any extra trainable parameters. LSTM also drastically reduces the number of training parameters. An LSTM can perform much better once the best hyperparameter setting is determined. 
</li>
</ol><br>
</div>
<div id = "div12">
<p><b>APPENDIX</b>
<p>Many different hyperparameter settings were tried to see what would result in the best model.
This was done by trial and error. The different hyperparameter is highlighted in red
Some of these are documented below:
<ol>
<li>Residual MLP <br>
<table>
<tr>
<td style="color:red"> number_steps_in  </td>
<td style="color:red"> 168</td>
</tr>
<tr>
<td> number_steps_out  </td>
<td> 24</td>
</tr>
<tr>
<td> Dense_1 neurons   </td>
<td>256</td>
</tr>
<tr>
<td> Dense_2 neurons  </td>
<td> 24</td>
</tr>
<tr>
<td> optimizer  </td>
<td>Adam</td>
</tr>
<tr>
<td> Initial learning rate   </td>
<td> 1e-4</td>
</tr>
<tr>
<td> loss   </td>
<td>Mean squared error</td>
</tr>
<tr>
<td> Epochs  </td>
<td> 300</td>
</tr>
</table> <br>
Mean MAE = 6.27 <br>
Mean MRSE = 7.33
</li></br>
<li>Residual MLP <br>
<table>
<tr>
<td style="color:red"> number_steps_in  </td>
<td style="color:red"> 240</td>
</tr>
<tr>
<td> number_steps_out  </td>
<td> 24</td>
</tr>
<tr>
<td> Dense_1 neurons   </td>
<td>256</td>
</tr>
<tr>
<td> Dense_2 neurons  </td>
<td> 24</td>
</tr>
<tr>
<td> optimizer  </td>
<td>Adam</td>
</tr>
<tr>
<td> Initial learning rate   </td>
<td> 1e-4</td>
</tr>
<tr>
<td> loss   </td>
<td>Mean squared error</td>
</tr>
<tr>
<td> Epochs  </td>
<td> 300</td>
</tr>
</table> <br>
Mean MAE = 9.85 <br>
Mean MRSE = 10.96
</li></br>
<li> MLP
<p>There are many different ways to calculate the number of neurons in the hidden layers.
One of the ideas is to use the following formula: neurons = sqrt(input_size *
final_output_size)
<table>
<tr>
<td> number_steps_in  </td>
<td> 72</td>
</tr>
<tr>
<td> number_steps_out  </td>
<td> 24</td>
</tr>
<tr>
<td style="color:red"> Dense_1 neurons   </td>
<td style="color:red">117</td>
</tr>
<tr>
<td style="color:red"> Dense_2 neurons  </td>
<td style="color:red"> 52</td>
</tr>
<tr>
<td> Dense_3 neurons  </td>
<td>24</td>
</tr>
<tr>
<td> optimizer  </td>
<td>Adam</td>
</tr>
<tr>
<td> Initial learning rate   </td>
<td> 1e-4</td>
</tr>
<tr>
<td> loss   </td>
<td>Mean squared error</td>
</tr>
<tr>
<td> Epochs  </td>
<td> 300</td>
</tr>
</table> <br>
Mean MAE = 6.65 <br>
Mean MRSE = 7.66
</li></br>
<li>MLP<br>
Adding an extra hidden layer can help improve the MAE as compared to the
previously mentioned MLP with a single hidden layer. However, this increases the number of trainable parameters. <br>
<table>
<tr>
<td> number_steps_in  </td>
<td> 72</td>
</tr>
<tr>
<td> number_steps_out  </td>
<td> 24</td>
</tr>
<tr>
<td style="color:red"> Dense_1 neurons   </td>
<td style="color:red">256</td>
</tr>
<tr>
<td style="color:red"> Dense_2 neurons  </td>
<td style="color:red"> 64</td>
</tr>
<tr>
<td> Dense_3 neurons  </td>
<td>24</td>
</tr>
<tr>
<td> optimizer  </td>
<td>Adam</td>
</tr>
<tr>
<td> Initial learning rate   </td>
<td> 1e-4</td>
</tr>
<tr>
<td> loss   </td>
<td>Mean squared error</td>
</tr>
<tr>
<td> Epochs  </td>
<td> 300</td>
</tr>
</table> <br>
Mean MAE = 5.85 <br>
Mean MRSE = 6.87
</li> <br>
<li>Residual MLP: Adding an extra hidden layer to the Residual MLP does not help improve the MAE. <br>
<table>
<tr>
<td> number_steps_in  </td>
<td> 72</td>
</tr>
<tr>
<td> number_steps_out  </td>
<td> 24</td>
</tr>
<tr>
<td style="color:red"> Dense_1 neurons   </td>
<td style="color:red">256</td>
</tr>
<tr>
<td style="color:red"> Dense_2 neurons  </td>
<td style="color:red"> 64</td>
</tr>
<tr>
<td> Dense_3 neurons  </td>
<td>24</td>
</tr>
<tr>
<td> optimizer  </td>
<td>Adam</td>
</tr>
<tr>
<td> Initial learning rate   </td>
<td> 1e-4</td>
</tr>
<tr>
<td> loss   </td>
<td>Mean squared error</td>
</tr>
<tr>
<td> Epochs  </td>
<td> 300</td>
</tr>
</table> <br>
Mean MAE = 7.89 <br>
Mean MRSE = 9.14
</div>





