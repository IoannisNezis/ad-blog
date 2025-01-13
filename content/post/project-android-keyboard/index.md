---
title: "Spelling Correction and Autocompletion for Mobile Devices"
date: 2021-03-11T21:24:44+01:00
author: "Ziang Lu"
authorAvatar: "img/project-android-keyboard/ziang.png"
tags: [android, keyboard, nlp, n-gram, PED]
categories: ["project"]
image: "img/project-android-keyboard/title_pic.jpg"
draft: false
---
A virtual keyboard is a powerful tool for smartphones, with which users can improve the quality and efficiency of the input. In this project, we will explore how to use n-gram models to develop an Android keyboard which gives accurate corrections and completions efficiently.
<!--more-->

# Content

1. <a href="#introduction">Introduction</a>
2. <a href="#AlgoDat">Algorithm and Data Structure for Similarity Calculation</a>
3. <a href="#N-gram Models">N-gram Models</a>
4. <a href="#Corpus">Corpus</a>
5. <a href="#App">App</a>
6. <a href="#Evaluation">Evaluation</a>
7. <a href="#Potential Improvements">Potential Improvements</a>
8. <a href="#Summary">Summary</a>

# <a id="introduction"></a> Introduction



An efficient keyboard will help user spare lot of work and time when inputting a text. For instance, we want to input a sentence as below:


<div style="text-align: center;">  <font color = "black"> <em> "We are going to watch a movie" </em> </font> </div> 

<br />


There are 29 characters (without spelling mistakes) to enter which means one needs to press keys for 29 times without any helpful function. However, if we could have a magical keyboard which shows us candidates for the next word and helps in correcting spelling mistakes given our current input , we may reduce a lot of steps. Generally, we expect such a scenario:
<br />

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font size="4"> current input:</font> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <i> Wee </i> <font size="4"><em>(expecting correction ) </em></font><br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font size="4"> candidates to choose:</font><code> We  Lee  Bee </code> <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font size="4"> current input:</font> <i>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; We </i>  <font size="4"><em>(expecting prediction) </em></font><br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font size="4"> candidates to choose:</font> <code>are  do  were</code> <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font size="4"> current input:</font> <i> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;We are g </i> <font size="4"><em>(expecting completion) </em></font><br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font size="4"> candidates to choose:</font> <code>going  gone  getting </code><br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font size="4"> current input:</font> <i> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;We are going</i> <font size="4"><em>(expecting prediction) </em></font><br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font size="4"> candidates to choose:</font> <code>to  by  on </code><br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ............

<br />
In the next sections, we will explore how to make such a “smart” keyboard with combination of n-gram model, Prefix Edit Distance (PED) and Q-gram Index and how well it works.

# <a id="AlgoDat"></a> Algorithm and Data Structure for Similarity Calculation
<br />
Assuming that we are going to type the word “_movie_”, but we have accidently typed “_movvie_". We hope that our keyboard could find this error quickly and show us the correct version of this word. Or maybe you feel so tired to input a very long word and hope your keyboard could guess your final goal by reading a prefix of a word which you have typed, for example, you want to type “_something_” but you only need to input “_somet_” and you will get a candidate “something” to choose. Now the question is, how could a keyboard measure the difference between a wrong word and correct one and the needed work to get to your final complete word? Namely, why could this keyboard show you a candidate with “_movie_” but not with “_move_”? Why “_anything_” should not be expected based on “_somet_”? To find answers, we will firstly take a look at **Prefix Edit Distance (PED)**.
<br>
<br>
<b> <font size="5" color="black"> Edit Distance (ED) </font></b>
<br>

**Definition** for two strings x and y. ED(x, y): = minimal number of transformations to get from x to y <br>
<br>
Transformations allowed are: <br>
<b>insert</b>(i, c):   insert character c at position i <br>
<b>delete</b>(i):    delete character at position i <br>
<b>replace</b>(i, c):  replace character at position i by c <br>

**Example:** <br>

$$ somethfinge  \stackrel{replace}{\longrightarrow}  somethinge  \stackrel{delete}{\longrightarrow}  something $$<br>
**ED** (somethfinge, something) = 2
<br>
<br>
With a keyboard we will always input a word by starting from the first letter on the left and going to the end on the right. Therefore, if we have inputted “_somet_” as part of word “_something_”, we will expect that “_something_” should be shown as the best candidate, but in this case, an unexpected candidate “_same_” with **ED** = 2 will be shown with a higher probability than “_something_” because of the **ED** = 5. For this reason, we extend Edit Distance into **Prefix Edit Distance**. <br>
<br>
<br>
<b> <font size="5" color="black"> Prefix Edit Distance (PED) </font></b>
<br>
<br>
**Definition** $$ \small PED (x, y) = min_{y'} (ED (x, y')) $$ where y' is a prefix of y. <br>

Given a string x, one task of keyboard is to find out all strings $$ \small y_{i}$$ so that $$ \small PED (x, y_{i}) \leq 2 $$ and return the result to user. <em> (For my keyboard the threshold of PED is 2, but it could be changed as one wishes) </em> <br>

A response time feels interactive until around <b>200ms</b>. It takes us a lot of time if we compare x with other words one by one from a dictionary. It makes calculation very inefficient and unnecessarily slow. 
<br>
<br>
For example, PED between “_movies_” and “_cinema_” is intuitively larger than 2 and it is unnecessary to make a calculation.

<br>

To filter out those “impossible” words, we use **q-Gram Index** to minimize the size of group of words which need to be compared.
<br>
<br>
<b> <font size="5" color="black"> q-Gram Index </font></b>
<br>
<br>

**Definition** Q-grams of a string are simply a set of all substrings of this string with a length q.
<br>

If q = 3, then 3-Grams of “_freiburg_” would be “_fre_”, “_rei_”, “_eib_”, “_ibu_”, “_bur_”, “_urg_”; 3-grams of “_movie_” are “_mov_”, “_ovi_”, “_vie_”.
<br>

To optimize the match, we will pad the q – 1 special symbols (we use $) in the beginning for PED (and for ED at both beginning and end).
<br>

Consider x and y with PED (x, y) = δ
Intuitively: if x and y are not too short, and δ is not too large, they will have one or more q-grams in common. So we could apply some rules to judge if calculation of PED should be executed on the number of q-grams in common.
<br>

**Example:**
<br>

x = freiburg  <br>
y = breiberg  <br>
**q** = 3, **δ** = 2 
<br>

after padding "_$$_" at the start, we get 3-grams of "_freiburg_" and "_breiberg_":
<br>
<br>
<font color="black"> <i> <code>„$$f“   „$fr“  „fre“  „rei“  „eib“  „ibu“  „bur“  „urg“</code></i></font>
<br>
<font color="black"> <i> <code>„$$b“  „$br“  “bre „ “rei“  „eib“  “ibe“  “ber“  „erg“ </code></i></font>
<br>
<br>
number of q-grams in common: 2.
<br>
Formally: let x' and y' be the padded versions of x and y.
<br>
Then it holds: $$ comm(x', y') \geq |x| – q ∙ δ $$
|x| = 8, |y| = 8, **δ** = 2, **q** = 3 <br>
Hence: comm (x’, y’) = 2 **≥** 8 – 3 * 2 = 2 fulfilled!<br>

Therefore, the formula could be applied to check if a comparison should be executed. <br>
<br>

**Example:**
<br>

x = freiburg <br>
 **q** = 3, **δ** = 2 (that means we expect the PED should be less than 3) <br>
<br>
y1 = freiberg:  5  &nbsp; |x| – 6 = 2 -> Yes <font color="green"> &radic; </font><br> 
y2 = nürnberg:  0   |x| – 6 = 2 -> No <font color="red">× </font><br> 
y3 = hamgurg:   1   &nbsp;|x| – 6 = 2 -> No <font color="red">× </font><br> 

<br>

So, for this example, we only have to compute **PED(freiburg, freiberg)** … which is 1, hence breiburg is an output as a match.
<br>
<br>
_Note: More details about Edit Distance and q-gram Index see in the lecture <a href="https://ad-wiki.informatik.uni-freiburg.de/teaching/InformationRetrievalWS1920"> InformationRetrieval </a>._
<br>
<br>

# <a id="N-gram Models"></a> N-Gram Models

We could find a lot of candidates which fulfill a given threshold of **PED** regarding a string x. But some candidates are evidently impossible to be the next word. For instance, you have typed a incomplete sentence “_who is th_”, and you may get some candidates such as “_those_”, “_than_” or “_thanks_”. These words seem not so convincing like “_there_” or “_that_”. Actually, if you have the same idea, that means, you are considering probabilities in this case. Hence, we still need a so-called n-gram model which could be applied to count and analyze probability of different combinations of words based on a corpus for a precise completion or prediction.
<br />
<br>
**Definition** An n-gram model is a sequence of N words: a 2-gram (or bigram) is a two-word sequence of words like “_we are_”, “_going to_”, or “_watch a_”, and a 3-gram (or trigram) is a three-word sequence of words like “_we are going_”, or “_are going to_”. 
<br />
We will see how to use those n-gram models to estimate the probability of the last word given the previous words.
<br>
<br>
Let’s consider a word **w** and a history **h** (start of the sentence) and a corpus **K**.
<br> <br>
<em>h: The weather is so good
<br> w: that </em> 
<br> <br>
we want to calculate the probability of the word <b>w </b> given the history <b> h, </b> and we denote it as <b> P (_that_ | _the weather is so good_)</b>. Since the size of corpus **K** is limited and some combination like <em>“today’s weather is so good”</em> may not appear, we will instead approximate the history just by the last few words. <br> <br>
In other words, instead of computing <b>P (_that_ | _the weather is so good_) </b>, we approximate it with the probability of bigram model: <b> P( _good_ | _that_ ). </b> In this case, we just need to count the frequency of sequence <em>“good that”</em> and frequency of <em>“good” </em> in the corpus K (here we denote that as <b>C(sequence)). </b>
$$ P(good|that) = \frac{C(good \ that)}{C(that)}$$
<br>
<b>C(_good that_) </b>is to count how many times the combination <em>“good that”</em> appears in the corpus. <b>C(_good_)</b> is to count how many times the word “_good_” in **K** appears. <br>To extend our bigram model to general n-model, we have this formula as follow:
$$ P(w_{n}|w_{n-N+1}^{n-1}) = \frac{C(w_{n-N+1}^{n-1}w_{n})}{C(w_{n-N+1}^{n-1})} $$
For instance, when we use a trigram model in the case above, we need to compute: 
$$ \frac{C(that|so \ good)}{C(so \ good)} $$
<br>
<br>
# <a id="Corpus"></a> Corpus
To train n-gram models, I used corpuses about web text and tweets from <a>https://www.nltk.org/howto/corpus.html</a>, which cover different topics such as positive, negative expressions, political discussions and so on. The raw corpus includes a great amount of emojis and special symbols which produces noise into n-gram models. Hence, I filtered out those emojis and symbols so that my n-gram models become “cleaner” and concentrate only on plain sentences.
<br> <br>
<center> Corpus Info </center>
<br>
<br> 

||__number of characters__|__number of words__|__number of sentences__|__number of documents__|__topics__|
| --- | --- | --- | --- | --- | --- |
|corpus from tweets|1264807|223201|50070|3|negative, positive and politic tweets|
|corpus from web   |1469355|255328|57425|4|firefox, overheard, singles, wine|

<center> Grams Info (after deleting some grams that appear only once or twice in corpus) </center>
<br>

|__gram__|__amount__|
|---|---|
|unigram|9295|
|bigram |21561|
|trigram|10091|

Before we have talked about the aim and construction of a **q-gram**. Given a word **w**, we need to find all words from a dictionary whose **PED** fulfills the threshold **delta**. To reduce the response time, we should compute **q-grams** of all words  in advance from the dictionary, and once a query is executed, we will compute the number of common grams between **w** and all other words from the dictionary. <br> <br>
To minimize the intern storage of app and its startup delay, the total number of words from a dictionary has been limited into 10000. Hence, for this dictionary I used 10000 most common used English words from <a> www.mit.edu/~ecprice/wordlist.10000 </a>. Another issue is that some words from corpus may not be included in the dictionary. Therefore, after keeping the words which appear both in dictionary and corpus, I removed 4400 words from the dictionary which never appeared in the corpus and added 4400 new words by their frequencies into the dictionary from the corpus.
<br>
# <a id="App"></a> App
<br><br>

<td><img src="/img/project-android-keyboard/screen_shoot1.jpg" style="width:346px;height:746px;text-align:center;margin: 0px;"/></td>

<br><br>


The keyboard is implemented in an Android App which could be used to test and evaluate the utility of keyboard.
<br><br>
<br>
<b> <font size="5" color="black"> App Design </font></b>
<br><br>
The basic routine is as follows, when any function above is triggerd: <br><br>
**1** Keyboard accepts user’s input &#8595;<br>
**2** Program splits input by punctuation&#8595;<br>
**3** The last part will be used to decide whether user expect a completion (correction) or prediction &#8595;<br>
**4** Through computation of PED and n-gram model’s probability program return maximal 3 candidates with highest probability&#8595;<br>
**5** User choose candidate <font color="grenn"> &radic; </font>
<br><br>
<center><b>Completion Routine</b></center>
<td><img src="/img/project-android-keyboard/completion_routine.png" style="width:900px;height:346px;text-align:center;margin: 0px;"/></td>

<br><br><br><br><br>


<center><b>Correction Routine </b> (same as completion, here I just want to show how this routine could also be applied to correct spelling mistakes</center>
<td><img src="/img/project-android-keyboard/correction_routine.png" style="width:900px;height:346px;text-align:center;margin: 0px;"/></td>

<br><br><br><br>

<center><b>Prediction Routine</b></center>
<td><img src="/img/project-android-keyboard/prediction_routine.png" style="width:900px;height:346px;text-align:center;margin: 0px;"/></td>

<br><br> <br>
<b> <font size="5" color="black"> Ranking of Candidates </font></b>
<br><br>
For the 4-th step, we need to consider additionally two issues. Some combination of words may never appear in a corpus and exception such as *ZeroDivisionError* may be raised in program. Therefore, we will apply a <b>flexible</b> n-gram model: final probability = <b> P (trigram) + P (bigram) + P (unigram)</b>. And final probability will be initialized as 0.0.
$$ \small P = \lambda 1 * P(w_{n}) \ (unigram \ probability) $$ 
$$ \small \ \ \ \ \ \ \ \ \ \ +  \lambda 2 * P(w_{n} | w_{n-1}) \ (bigram \ probability) $$
$$ \small \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ + \lambda 3 * P(w_{n} | w_{n-2, \ n-1}) \ (trigram \ probability) $$

&#955;1, &#955;2 and &#955;3 are so-called **weights**, which need to be tuned through experiments. Generally, &#955;3 will be assigned with a relatively larger value because the longer sequence as history will give a more precise result and should play a decisive role. And in this project, I will set &#955;1 to 0.1, &#955;2 to 0.3 and &#955;3 to 0.6.
<br><br>
The issue is that sometimes we may encounter a such case:
<br><br>
Threshold = 2 **(PED)** <br>
count of “_support_” in corpus:   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<code>10000 </code><br>
count of “_scotland_” in corpus:  &nbsp;&nbsp;&nbsp;&nbsp; <code>8000 </code><br>
count of “_should_” in corpus:    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  <code>9000</code><br>
count of “_some_” in corpus:      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <code>5000</code><br>
count of “_something_” in corpus: &nbsp;&nbsp;<code>3000</code><br>
<br>
When we type “_som_” at the beginning of a text, program will give us all words with **PED** <= 2 and return three of them by their probabilities. Only regarding the probability of unigram (start of a sentence), we will get a list which consists of “_support_”, “_should_” and “_scottland_”, beacuse they appear at a very high freqency. Intuitively, user may expect a list consisting of “_some_” and “_something_” given the incomplete part “_som_”. Therefore, we need to “punish” those words like “_support_” and “_scottland_” because of their larger PED than “_something_” or “_some_”.<br><br>
To filter out those words furtherly which we do not want to see, we could define the final probability of a candidate **P** as follow, here we denote original probability based on a n-gram model as **P<SUB>n** , a punishment weight as <b> &#945; </b> and **PED** keeps its original meaning: <br>
$$  P = P_{n} - \alpha * PED. $$

Probability of words with larger **PED** will be reduced because of the punishment weight <b> &#945; </b> and it is just what we want. Hence, candidates that user really want would be returned correctly. (In the case above, “some” and “something” would be seen with higher probability). <br>Evaluation below will show how the specific value of punishment value will influence the result and will also prove this method works well.

# <a id="Evaluation"></a> Evaluation
In the evaluation the utility of autocorrection and completion will be tested respectively in my keyboard denoted as *ZKeyboard* and the *Android API 30* system keyboard.<br><br>

<b> <font size="5" color="black"> Design of Evaluation </font></b>
<br><br>
**95%** of corpus content will be used as the trainset and be applied to build n-gram models. Respective **5%** of the content in web and tweets will be used to evaluate my keyboard with an automatic program. In addition, I picked 100 sentences respectively from web corpus and tweets corpus so that I can manually evaluate the *API 30 keyboard* and my *ZKeyboard*. The evaluation **criterium** is percentage of saved steps or pressed keys by using the keyboard, assuming we need to type every letter for once without a helpful keyboard. The pressed keys used to change capital mode or go to find symbols will not be counted in the total number because the project does not focus on the design and layout of a keyboard but only the models and algorithms.<br>

Firstly, I will try to assign different value to punishment weight in the test of completion based on the 5% of corpus. That could show if this method really works and what the proper value should be.<br>

For test of autocorrection, I will change the first letter of every word whose length is larger than 1 from the 5% of corpus and the 200 sentences. Generally, if we find one spelling mistake in a word, we need two steps to correct it (delete and insert). Hence, in this case, we need to increase total steps that are needed without helpful keyboard.<br>


At last, I want to show how the trainset from a corpus could adapt for different text’s type. For this purpose, I will use trainset from 95% of web to evaluate 5% of tweets and 95% of tweets to evaluate 5% of web.<br>

<b> <font size="5" color="black"> Results </font></b>
<br><br>

*Evaluation 1: looking for the best punishment weight &#945; and see if it really works.* <br>
*Train set: 95% from web + tweets* <br>
*Test set: 5% from web*<br>
<br>


| __ALPHA (punishment)__ | __Reduced steps in web (5%)__ |
| --- | --- |
| 0.0   |27.10%|
| 0.0005|36.04%|
| 0.005 |41.16%|
| 0.05  |41.20%|
| 0.5   |41.20%|
| 1.0   |41.20%|

<br>
<p>
Before we have talked about how the punishment value alpha could help filtering out those words which are evidently different from the goal word in the sense of PED. Through experiments it proves that  when the value of alpha is equal or larger than 0.5, the utility of the keyboard will be maximized. Therefore, for the next evaluation the alpha will applied as 1.0.	
</p>

<br>
<br>

<em> Evaluation 2: evaluate function of autocompletion </em> <br>
*Train set: 95% from web + tweets* <br>
*Test set: 5% contents from web, 5% contents from tweets, 100 sentences from 5% of web, 100 sentences from 5% of tweets* <br><br>


|         |__Web(small)__ | __Web (5%)__ | __Tweets(small)__|__Tweets(5%)__|
| ---     | ---           | ---          |---               | ---          |
|__API30__    |43.19%         |        --     |40.93%            |--        |
|__ZKeyboard__|43.00%         |41.20%        |43.72%            |38.59%        |

<br>
<p>
ZKeyboard behaves better than API30 keyboard when tested by 100 sentences from tweets. The perfomance's difference between API30 keyboard and Zkeyboard is very tiny when they were tested by 100 sentences from web. 200 sentences from tweets and web are those sentences which look more normal in the sense of syntax and grammar, so Zkeyboard has a better performance on them. 
</p>
<br>
<br>

<em>Evaluation 3: evaluate function of autocorrection <br>
Train set: 95% from web + tweets<br>
Test set: 5% contents from web, 5% contents from tweets, 100 sentences from 5% of web, 100 sentences from 5% of tweets </em> <br><br>
<br>

|             |__Web(small)__ | __Web (5%)__ | __Tweets(small)__|__Tweets(5%)__|
| ---         | ---           | ---          |---               | ---          |
|__API30__    |  21.00%       |        --    |     18.60%       |--            |
|__ZKeyboard__|  21.35%       |    23.62%    |     24.06%       |   20.71%     |

<br>
<p>
For the evaluation of autocorrection, the first letter of every word whose length larger than 1 would be changed. ZKeyboard does better than API30 keyboard especially in the test dataset from tweets. One reason for that: political part from tweets includes a lot of special named entities which may not be identified by the API30 keyboard. In contrast, in training stage, n-gram model that Zkeyboard uses has remebered thoes special named entities which appear at higher frequency, therefore, ZKeyboard has a better performance.
</p>
<br>

<em> Evaluation 4: evaluate adaptability of n-gram model trained by a particular corpus by the means of evaluating function of autocompletion </em> <br>
*Train set: 95% from web + tweets* <br>
*Test set: 5% contents from web, 5% contents from tweets, 100 sentences from 5% of web, 100 sentences from 5% of tweets* 
<br>

|                 | __web (5%)__ |__tweets(5%)__|
| --- | --- | --- |
| __95% web__         |41.62%        |31.33%        |
| __95% tweets__      |33.83%        |39.27%        |
| __95% tweets + web__|41.20%        |38.59%        |

<br>
<p>
The result shows that when we apply n-gram model trained by corpus A to evaluate corpus B (with different sources), the accuracy could be reduced. One can also observe that the performance's difference of n-gram model trained by corpus from tweets is much smaller than that of n-gram model trained by corpus web. The reason is not clear but one possibility is that the content of tweets is more diversified than that of web.  
</p>
<br>

# <a id="Potential Improvements"></a> Potential Improvements

An In-App keyboard could be only used to evaluate the keyboard. In the next stage, I will implement language models and algorithms on a system keyboard with which people could use to input text in any edit view on a mobile device. <br>

More grams. Now keyboard uses a flexible n-gram model (maximal trigram) to calculate probability. In the future, I will extend my n-gram model into 5 or 6 gram so that the accuracy will be improved.  <br>

Using database to accelerate startup speed. Currently, I store data (model, grams info) in .txt format so that app should read them at the startup. And it takes app about 2 seconds to read the data every time when app starts (for first installation it may take longer). This has limited the size of data set and makes startup of keyboard a little bit slow. In the next stage, I will use dataset to store all data and reconstruct my program so that no data needs to be read at the startup. Interaction with data will only be needed on queries.<br>

The keyboard could not follow grammatical rules to filter out unsuitable candidates. This problem should be solved by POS-Tagging to improve grammatical accuracy of keyboard.<br>

At last, keyboard should memorize user’s input so that most common typed words by users will have a higher priority than others.<br>

# <a id="Summary"></a> Summary

With the help of n-gram model, Prefix Edit Distance and q-gram Index, we have developed such a smart keyboard (ZKeyboard) which could give relatively accurate corrections and completions. And compared with API30 keyboard, Zkeyboard does not bad not only in completion but also in spelling correction. But we still have seen many aspects which need to be improved such as ignored grammar rules, limit of storage, accuracy of n-gram model and so on. To make a keyboard give more accurate corrections and completions efficiently, we need more complex language models and do everything potential to improve the performance of the keyboard.
