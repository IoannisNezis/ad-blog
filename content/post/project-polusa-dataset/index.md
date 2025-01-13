---
title: "POLUSA: A Large Dataset of Political News Articles"
date: 2020-06-17T10:40:04+02:00
author: "Lukas Gebhard"
authorAvatar: "img/project-polusa-dataset/avatar-lukas-gebhard.png"
tags: ["corpus", "nlp", "text mining", "computational social sciences"]
categories: ["project"]
image: "img/project-polusa-dataset/header.png"
draft: false
---

I present POLUSA, a dataset of 0.9M online news articles covering policy topics. POLUSA aims to represent the news landscape as perceived by an average US news consumer. In contrast to previous datasets, POLUSA allows to analyze differences in reporting across the political spectrum, an essential step in, e.g., the study of media effects and causes of political partisanship.

<!--more-->

*This article is based on a [poster presented at JCDL'20](https://doi.org/10.1145/3383583.3398567).*

# Contents

1. <a href="#intro">Introduction</a>
1. <a href="#creation">Dataset Creation</a>
    1. <a href="#base-selection">Base Selection</a>
    1. <a href="#near-duplicate-detection">Near-Duplicate Detection</a>
    1. <a href="#selection-articles">Selection of English News Articles Covering Policy Topics</a>
    1. <a href="#temporal-balancing">Temporal Balancing</a>
    1. <a href="#balancing-popularity">Balancing by Outlet Popularity</a>
    1. <a href="#political-leanings">Assignment of Political Leanings</a>
1. <a href="#characteristics">Dataset Characteristics</a>
    1. <a href="#outlet-distribution">Distribution of Outlets</a>
    1. <a href="#leaning-distribution">Distribution of Political Leanings</a>
    1. <a href="#temporal-distribution">Temporal Distribution</a>
    1. <a href="#length-distribution">Distribution of Article Lengths</a>
1. <a href="#limitations">Limitations</a>
1. <a href="#conclusion">Conclusion</a>

# <a id="intro"></a> Introduction

News articles serve as a crucial source of information in various disciplines, e.g., [in the social sciences and digital humanities scientists analyze news coverage](https://www.doi.org/10.1109/JCDL.2019.00036) to understand societal issues and trends. A common requirement is that the analyzed news articles reflect the news landscape, e.g., with respect to the political spectrum but also the number of readers per outlet contained in the dataset. In computer science, articles are commonly used, e.g., to train machine learning algorithms, recently – with the rise of deep learning – [requiring very large amounts of data](https://arxiv.org/abs/1907.11692).

However, the creation of such datasets requires significant effort, and existing news datasets suffer from at least one of the following shortcomings: 

- They do not represent real-world distributions of news, e.g., what the average news consumer would typically read ([Media Frames Corpus](http://aclweb.org/anthology/P15-2072), [AllSides.com Dataset](http://aclweb.org/anthology/W18-6509), [NewsWCL50](https://ieeexplore.ieee.org/document/8791197/), [NELA-GT-2018](http://arxiv.org/abs/1904.01546)).
- They require considerable preprocessing effort, e.g., because they are inefficiently accessible website dumps and contain duplicates and noise ([CC-News](http://arxiv.org/abs/1907.11692), [RealNews](http://arxiv.org/abs/1905.12616)).
- They lack an association of outlets or articles to their political leanings ([Media Frames Corpus](http://aclweb.org/anthology/P15-2072), [NewsWCL50](https://ieeexplore.ieee.org/document/8791197/), [CC-News](http://arxiv.org/abs/1907.11692), [RealNews](http://arxiv.org/abs/1905.12616)), a common requirement in the social sciences, or the process of deriving such associations is non-transparent ([AllSides.com Dataset](http://aclweb.org/anthology/W18-6509)).

To address these shortcomings, I present POLUSA, a dataset that aims to represent the landscape of online news coverage as perceived by an average US news consumer. The dataset contains 0.9M news articles covering policy topics published between Jan. 2017 and Aug. 2019 by 18 news outlets representing the political spectrum. Using a systematic aggregation of previously published measures, I label each outlet by its political slant. The dataset is balanced with respect to publication date and outlet popularity.

POLUSA is [available at Zenodo](https://doi.org/10.5281/zenodo.3813663).

# <a id="creation"></a> Dataset Creation

To create POLUSA, I perform a series of six steps, which aim to ensure the characteristics described in the <a id="intro">introduction</a> and to increase the quality of the dataset: (1) base selection, (2) near-duplicate removal, (3) selection of English news articles covering policy topics, (4) temporal balancing, (5) balancing by outlet popularity, and (6) assignment of political leanings.

## <a id="base-selection"></a> Base Selection

As input data, I use the [CommonCrawl news archive](https://commoncrawl.org/2016/10/news-dataset-available/) (CCNA), a large set of news websites collected as part of the CommonCrawl project. [Felix Hamborg](https://www.isg.uni-konstanz.de/people/doctoral-researchers/felix-hamborg/) (University of Konstanz) downloaded CCNA in September, 2019. Using [news-please](https://github.com/fhamborg/news-please), he extracted potential news articles and metadata from the webpages, and kindly grants me access to the output.

Because CCNA lacks data for various timeframes and news outlets, I need to select a subset of articles that is as large as possible while having an as consistent as possible number of articles for any given timeframe within. As an additional constraint, I disregard news outlets that do not publish any straight news covering US politics written by professional journalists. The resulting subset contains all articles of 30 news outlets, published between January 2017 and August 2019. After removing exact duplicates, the *base selection* contains 3.6M articles.

## <a id="near-duplicate-detection"></a> Near-Duplicate Detection

As a second step, I remove near-duplicates. For each article, I first hash all token-level tri-grams. Then, I compute a [simhash](http://portal.acm.org/citation.cfm?doid=1242572.1242592) of the resulting vector of hashes. For each outlet, I cluster the obtained simhashes using a simple greedy algorithm:

For a given outlet, let \\(A\\) be the set of all articles published by that outlet. Further, let \\(d(a, b)\\) be the number of bits by which the simhashes of articles \\(a, b \in A\\) differ. For some threshold \\(k\\), define

$$S\_k(a) = \\{ b \in A \;|\; d(a, b) \leq k\\}$$

to be the near-duplicates of \\(a\\) and

$$S\_k = \bigcup\_{a \in A, \\\\ |S\_k(a)| > 1} S\_k(a)$$

the set of articles having at least one near-duplicate. As long as \\(S\_k \neq \emptyset\\), iteratively pick some \\(a \in S\_k\\), and extract a new cluster \\(C\_k(a) = S\_k \cap S\_k(a)\\) from \\(S\_k\\), followed by assigning \\(S\_k \leftarrow S\_k - C\_k(a)\\).

After having experimented with threshold \\(k\\), I set it to \\(k := 9\\). Finally, I remove all but the newest article for each cluster.

This way, I remove 5 % of articles from the base selection, mostly consisting of outdated versions that resulted from minor article revisions, e.g., word insertions or corrections of numbers.

As an example, here are two versions of an article. My procedure correctly identifies the first one as a near duplicate of the second one. Passages that only occur in the respective document but not the other are highlighted in red; skipped passages are identical.

<table>
<tr>
<th>Article by M. Rose, published by Reuters on 09/04/2019 at <span style="color:red">4:11pm</span></th>
</tr>
<tr>
<td><b>France sets conditions for another Brexit delay before May arrives</b>
<br>
<em>French President Emmanuel Macron's office set out blunt conditions on Tuesday as <span style="color:red">British</span> Prime Minister Theresa May headed for Paris to ask him to agree to a delay in Britain's departure from the European Union.</em>
<br>
PARIS/BERLIN (Reuters) - French President Emmanuel Macron’s office set out blunt conditions on Tuesday as <span style="color:red">British</span> Prime Minister Theresa May headed for Paris to ask him to agree to a delay in Britain’s departure from the European Union.
<br>
[Skipping 243 characters]
<br>
As her ministers held crisis talks in London with the <span style="color:red">opposition</span> Labour Party in the hope of breaking the domestic deadlock, May dashed to Berlin and then Paris on the eve of Wednesday’s emergency EU summit.
<br>
[Skipping 2102 characters]
<br>
British Prime Minister Theresa May <span style="color:red">is welcomed by German Chancellor Angela Merkel, as they meet</span> to discuss Brexit, at the <span style="color:red">chancellery</span> in <span style="color:red">Berlin, Germany</span>, April 9, 2019. REUTERS/<span style="color:red">Fabrizio Bensch</span>
<br>
In London, <span style="color:red">British</span> Solicitor General Robert Buckland said May would “listen carefully” to any constructive suggestions made by the EU on the length of the extension, and conceded that the government might not have managed to ratify an exit deal in parliament before European elections are held on May 23-26.
<br>
[Skipping 1181 characters]
<br>
McDonnell said a customs union with the EU, seen as the most likely area for compromise but so far resisted by May’s government, was the first item on the agenda for the talks, which were to include <span style="color:red">finance minister</span> Philip Hammond.
<br>
Slideshow (<span style="color:red">15</span> Images)
<br>
The idea of a softer Brexit is anathema to eurosceptics in May’s Conservative party who have helped to defeat her divorce deal three times this year.
<br>
Meanwhile in London, <span style="color:red">lawmakers</span> were due to debate May’s Brexit delay proposal.
<br>
Without an extension, Britain is due to leave the EU at 2200 GMT on Friday, with no transition arrangements to cushion the economic shock.</td>
</tr>
</table>

<table>
<tr>
<th>Article by M. Rose, published by Reuters on 09/04/2019 at <span style="color:red">4:19pm</span></th>
</tr>
<tr>
<td><b>France sets conditions for another Brexit delay before May arrives</b>
<br>
<em>French President Emmanuel Macron's office set out blunt conditions on Tuesday as Prime Minister Theresa May headed for Paris to ask him to agree to a delay in Britain's departure from the European Union.</em>
<br>
PARIS/BERLIN (Reuters) - French President Emmanuel Macron’s office set out blunt conditions on Tuesday as Prime Minister Theresa May headed for Paris to ask him to agree to a delay in Britain’s departure from the European Union.
<br>
[Skipping 243 characters]
<br>
As her ministers held crisis talks in London with the Labour Party in the hope of breaking the domestic deadlock, May dashed to Berlin and then Paris on the eve of Wednesday’s emergency EU summit.
<br>
[Skipping 2102 characters]
<br>
<span style="color:red">French President Emmanuel Macron welcomes</span> British Prime Minister Theresa May <span style="color:red">as she arrives for a meeting</span> to discuss Brexit, at the <span style="color:red">Elysee Palace</span> in <span style="color:red">Paris, France</span>, April 9, 2019. REUTERS/<span style="color:red">Philippe Wojazer</span>
<br>
In London, Solicitor General Robert Buckland said May would “listen carefully” to any constructive suggestions made by the EU on the length of the extension, and conceded that the government might not have managed to ratify an exit deal in parliament before European elections are held on May 23-26.
<br>
[Skipping 1181 characters]
<br>
McDonnell said a customs union with the EU, seen as the most likely area for compromise but so far resisted by May’s government, was the first item on the agenda for the talks, which were to include <span style="color:red">Chancellor</span> Philip Hammond.
<br>
Slideshow (<span style="color:red">18</span> Images)
<br>
The idea of a softer Brexit is anathema to eurosceptics in May’s Conservative party who have helped to defeat her divorce deal three times this year.
<br>
Meanwhile in London, <span style="color:red">MPs</span> were due to debate May’s Brexit delay proposal.
<br>
Without an extension, Britain is due to leave the EU at 2200 GMT on Friday, with no transition arrangements to cushion the economic shock.</td>
</tr>
</table>

## <a id="selection-articles"></a> Selection of English News Articles Covering Policy Topics

Next, I select English news articles covering policy topics. This amounts to three steps:

First, I use [Nakatani's language classifier](https://github.com/shuyo/language-detection/blob/wiki/ProjectHome.md) to detect the articles' languages. I drop all non-English articles, which make up 6 % of the base selection.

Second, I remove non-article content. To do so, I use manually derived URL heuristics. For example, the URL `http://example.com/gallery/b-spears.html` likely links to a gallery of photos. Using a blacklist of 47 URL segments, e.g., `/gallery/`, I identify 6 % of the base selection as non-article content.

Third, I filter out non-political news such as sports, weather, and entertainment articles. I use a broad characterization of politics: Politics is about ["who gets what, when, and how"](https://www.cambridge.org/core/journals/american-political-science-review/article/politics-who-gets-what-when-how-by-harold-d-lasswell-new-york-whittlesey-house-1936-pp-ix-264/90C407BEDE6963B3D2C84FF79C695E1E). With that characterization, I consider some business news or tech news as political, depending on actual contents. As the articles do not have category tags, I again fall back to URL heuristics using a second blacklist of 60 URL segments, e.g., `/weather/` and `/sports/`. This way, I discard 13 % of the base selection.
To increase the political filtering performance, I train a convolutional neural network using [GloVe](https://www.aclweb.org/anthology/D14-1162/) word embeddings on a labeled set of 0.6M articles, extracted from CCNA, the [HuffPost dataset](https://www.kaggle.com/rmisra/news-category-dataset) and the [BBC dataset](http://mlg.ucd.ie/datasets/bbc.html). I fit the classifier on 87.5 % of the data. My evaluation on 12.5 % of the data yields F1=94.4 (p=95.6, r=93.2). The following histogram shows the distribution of predicted probabilities across all articles in the base selection. Most of the probabilities are around 0 % or 100 %, so in most cases the classifier is very confident in its decision whether an article is political or not:

<img src="/../../img/project-polusa-dataset/politicalness_distribution.png" title="Distribution of Estimated Probabilities"></img>

As a sanity check, I average the estimated probabilities over several subsets of the base selection:

Article Filter | Estimated Average Probability (%)
--- | ---
URL contains `/politics/` | 97
URL contains `/business/` | 69
URL contains `/sports/` | 3
Fulltext contains 'White House' | 93
Fulltext contains 'Britney Spears' | 10

The values reassure that the classifier works as expected. It classifies almost all articles with URL segment `/politics/` as political articles, whereas the opposite is the case for articles with URL segment `/sports/`. Furthermore, 69 % of articles having `/business/` in the URL are classified political. This coheres with the working definition of politics, according to which business topics tend to have a political character. Moreover, according to the classifier, almost all articles containing 'White House' are political articles. If an article contains 'Britney Spears', however, it covers policy topics by only 10 % on average.

Finally, I only keep articles \\(a\\) that likely report on policy topics (\\(p(a) \geq 0.75\\)). This applies to 49 % of the base selection.

The trained classifier is [available at GitHub](https://github.com/lukasgebhard/Political-News-Filter).

## <a id="temporal-balancing"></a> Temporal Balancing

After applying the filtering steps described in the previous subsections, 1.6M articles remain. The following heatmap visualizes the distribution of these articles by publication date (rows) and outlet (columns). For each outlet, the distribution is normalized to the interval \\([0, 1]\\).

<img src="/../../img/project-polusa-dataset/temporal_distribution_per_outlet.png" title="Temporal Distributions on the Level of Outlets"></img>

Strikingly, some of the columns reveal gaps spanning several months. This is likely due to technical issues during the web crawling process for CCNA. To avoid temporal distortions in the dataset, I drop outlets with high temporal variation. I measure temporal variation by the [coefficient of variation](https://en.wikipedia.org/wiki/Coefficient_of_variation) \\(c_v\\), and drop outlets with \\(c_v > 0.95\\). This affects (0.2M articles of) 10 outlets: The Daily Caller, National Review, Townhall, Yahoo! News, The Washington Post, PBS, MSNBC, Bloomberg News, The State, and ThinkProgress.

## <a id="balancing-popularity"></a> Balancing by Outlet Popularity

POLUSA aims to represent the landscape of online news as perceived by an average US news consumer. Therefore, each outlet’s share of articles should depend on its popularity. Otherwise, any unpopular outlet with a high publishing rate would contribute overly many articles to POLUSA. Similarly, any popular outlet would be underrepresented if its publishing rate is comparably low.

To avoid such misrepresentation, I balance the outlets by popularity. As POLUSA is exclusively about online news, it would make sense to model an outlet's popularity based on the traffic on its web domain. However, raw traffic metrics are not publicly available in general. Therefore, I approximate an outlet's popularity by its Alexa rank. The Alexa rank is a rank of top-level domains [based on a combination of unique visitors and page views in the past three months](https://support.alexa.com/hc/en-us/articles/200449744-How-are-Alexa-s-traffic-rankings-determined-).

Given all articles that have not been filtered out in the previous subsections, the following plots show each outlet's article count versus its Alexa rank.

<img src="/../../img/project-polusa-dataset/article_count_by_alexa.png" title="Article Counts by Alexa Rank"></img>

Clearly, the left plot reveals three outliers: Reuters in the top left corner and Mother Jones and The Nation in the bottom right. In the right plot, the three outliers are disregarded. It suggests a weak linear relationship between article count and Alexa rank. For simplicity, I model an outlet’s article count as a linear function of its Alexa rank. I employ a linear regression model

$$ E(Y|X=x) = ax + b $$

where article count \\(Y\\) is a normally distributed variable that linearly depends on Alexa rank \\(X\\). Disregarding the three outliers, the ordinary least squares method estimates a slope of \\(a\\) = -57K, an intercept of \\(b\\) = 60K, and a residual standard error of 22K. The solid blue line in the above figures is the estimated regression line. The dashed lines confine a band that contains 95 % of the population of outlets, given the model.

To balance article counts, I randomly resample articles of outlets that lie outside of that band. Visually speaking, the procedure vertically shifts points into the band, more precisely, onto the nearest dashed line. For outlets are affected: The three outliers and The Guardian. The latter is just above the band's upper edge (the upper dashed line). None of the outlets lie beneath the band. Therefore, balancing the article counts amounts to subsampling articles of the four overrepresented outlets. That way, the procedure drops 8 % of articles of The Guardian, 83 % of Reuters, and 100 % of Mother Jones and The Nation.

## <a id="political-leanings"></a> Assignment of Political Leanings

As a final step, I assign political leanings to news outlets. To ensure reliable ratings, I systematically aggregate eight data sources from prior work -- self-declarations by outlets, results of content analyses by social scientists, and news consumer data from surveys and social networks:

| Publication              | Data source                                                                                                                         |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| [Niculae et al.](https://dl.acm.org/doi/10.1145/2736277.2741688)      | "Declared Conservative" and "Declared Liberal" in Table 3                                                                           |
| [Budak et al.](https://papers.ssrn.com/abstract=2526461)       | Figure 1(a)                                                                                                                         |
| [Budak et al.](https://papers.ssrn.com/abstract=2526461)        | "Opinion" value in Figure 1(b)                                                                                                      |
| [Baum and Groeling](http://www.tandfonline.com/doi/abs/10.1080/10584600802426965)   | Table 3 with "post-election and election-related stories excluded"; significance level 10 %                                         |
| [Mitchell and Weisel](https://www.journalism.org/2014/10/21/political-polarization-media-habits/)  | Figure "Ideological Placement of Each Source’s Audience"                                                                            |
| [Gentzkow and Shapiro](http://www.nber.org/papers/w15916) | "Share Conservative" in Table 2                                                                                                     |
| [Bakshy et al.](http://www.sciencemag.org/cgi/doi/10.1126/science.aaa1160)        | "Alignment" in file "top500" of the supplementary material; In case of multiple matching domains, I choose the most general one    |
| [Ribeiro et al.](https://pure.mpg.de/pubman/faces/ViewItemOverviewPage.jsp?itemId=item_3038810)       | "Political Bias" shown on the web interface; In case of multiple matching domains, I choose the most general one                   |

I derive aggregated ratings as follows:

1. *Categorization.* For each of the numerical data sources, I symmetrically shrink the value range as far as possible without discarding any data points. I divide the obtained interval into five equally-sized partitions. The union of the first two partitions corresponds to left-leaning outlets (`LEFT`), the center partition corresponds to the political center (`CENTER`), and the union of the last two partitions correspond to right-leaning outlets (`RIGHT`). (Note: After carefully inspecting the data sources, I concluded that they are too noisy to use a finer categorization granularity than `LEFT`-`CENTER`-`RIGHT`).
2. *Aggregation.* I define *agreement level* \\(a(o, p)\\) to be the share of data sources for which outlet \\(o\\) is assigned political leaning \\(p\\). I label \\(o\\) by \\(\mathit{argmax}_p a(o, p)\\) if \\(\mathit{max}_p a(o, p) \geq k\\). Otherwise, I label it `UNDEFINED`. I set \\(k := 0.75\\) to ensure a high level of agreement among the data sources, thus avoiding controversial assignments of political leanings.

The below table shows the obtained ratings for 37 outlets, including all outlets contained in POLUSA:

| Rating      | Outlets                                                                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `LEFT`      | AlterNet, HuffPost, Los Angeles Times, Mother Jones, NPR, PBS, Slate, The Economist, The Guardian, The Nation, The New York Times, ThinkProgress |
| `CENTER`    | ABC News, CBS News, NBC News, Reuters, USA Today, Yahoo! News                                                                                    |
| `RIGHT`     | Breitbart, CNS News, Fox News, National Review, The Blaze, The Daily Caller, The State, The Weekly Standard, Townhall                            |
| `UNDEFINED` | AOL, BBC, Bloomberg News, Chicago Tribune, CNN, Politico, MSNBC, Reason, The Wall Street Journal, The Washington Post                            |

# <a id="characteristics"></a> Dataset Characteristics

This section gives some basic insights into the data. Each item in POLUSA represents a news article with the following attributes:

| Attribute           | Description                                                                       |
| ------------------- | --------------------------------------------------------------------------------- |
| `date_publish`      | The date and, if available, time of publishing.                                   |
| `outlet`            | The name of the news outlet that published the article.                           |
| `headline`          | The headline.                                                                     |
| `lead`              | The lead paragraph.                                                               |
| `body`              | The main text.                                                                    |
| `authors`           | The authors.                                                                      |
| `domain`            | The web domain under which the article was published.                             |
| `url`               | The URL under which the article was published.                                    |
| `political_leaning` | The outlet's political leaning (see <a href="#political-leanings">Subsection 2.6</a>). |

I distinguish two versions of POLUSA:

- *POLUSA-Unbalanced* prioritizes quantity over quality. Dataset size is highest priority, even if this comes at the cost of an inaccurate representation of the news landscape. POLUSA-Unbalanced results from omitting the balancing steps (see <a href="#temporal-balancing">Subsection 2.4</a> and <a href="#balancing-popularity">Subsection 2.5</a>). The dataset has 1.6M articles.
- *POLUSA-Balanced* prioritizes quality over quantity. Here, the main goal is to represent the perceived landscape of political online news in the USA from January 2017 to September 2019 as accurately as possible, even if this means many news articles have to be dropped due to balancing steps. POLUSA-Balanced is the result of carrying out all of the steps described in <a href="#creation">Section 2</a>. This yields a dataset of 0.9M articles.

## <a id="outlet-distribution"></a> Distribution of Outlets

The following pie charts contrast the outlet distribution in POLUSA-Unbalanced (left chart) with that of POLUSA-Balanced (right chart). Outlets with less than 50K articles are summarized as "others". In POLUSA-Unbalanced, approximately one third of all articles belong to Reuters. In POLUSA-Balanced, the largest shares are held by The Guardian (11 %), Fox News (10 %) and Reuters (10 %).

<img src="/../../img/project-polusa-dataset/outlet_distributions.png" title="Distribution of Outlets"></img>

## <a id="leaning-distribution"></a> Distribution of Political Leanings

The following tables show the distributions of outlets and articles across the political spectrum. For both versions of POLUSA, the distributions are skewed to the left. For example, POLUSA-Balanced features thrice as many left-wing outlets (6) as right-wing outlets (2). Nevertheless, each of the three political sectors still contributes a significant share of articles.

POLUSA-Unbalanced | `LEFT` | `CENTER` | `RIGHT` | `UNDEFINED` | Total
--- | --- | --- | --- | --- | ---
Outlet count | 10 | 6 | 6 | 8 | 30
Article share | 20 % | 46 % | 11 % | 23 % | 100 %

POLUSA-Balanced | `LEFT` | `CENTER` | `RIGHT` | `UNDEFINED` | Total
--- | --- | --- | --- | --- | ---
Outlet count | 6 | 5 | 2 | 5 | 18
Article share | 31 % | 27 % | 16 % | 26 % | 100 %

## <a id="temporal-distribution"></a> Temporal Distribution

The following two histograms show the temporal distributions of article publications in POLUSA-Unbalanced (left histogram) and POLUSA-Balanced (right histogram). In POLUSA-Unbalanced, the article publication count increases slightly as time moves on. This increase is due to the temporal gaps in CCNA, as pointed out in <a href="#temporal-balancing">Subsection 2.4</a>. In contrast, the distribution is approximately uniform in POLUSA-Balanced, due to temporal balancing.

<img src="/../../img/project-polusa-dataset/temporal_distributions.png" title="Temporal Distribution"></img>

## <a id="length-distribution"></a> Distribution of Article Lengths

Finally, the following two histograms show the distribution of article lengths in POLUSA-Balanced. A typical article has about 400 to 4,000 characters. Interestingly, there are spikes at around 500 and 1,000 characters. Perhaps, some editorial departments desire articles to be of those lengths.

<img src="/../../img/project-polusa-dataset/length_distribution_v1.png" title="Distribution of Article Lengths"></img>

# <a id="limitations"></a> Limitations

POLUSA is not perfect:

- Due to gaps in the underlying CCNA (see <a href="#temporal-balancing">Subsection 2.4</a>), some outlets (e.g., Yahoo! News, Chicago Tribune) are underrepresented or not represented at all.
- I did not balance the selection of articles geographically or across the political spectrum.
- The procedure of extracting news articles from websites (see <a href="#base-selection">Subsection 2.1</a>) caused some flaws. For example, paginated articles were not merged together. As a result, an article spanning three pages wrongly appears as three seperate articles in POLUSA. As another example, if no lead paragraph was detected for an article, `lead` was set to be the beginning of `body`. That is, the first one or two sentences of many articles were duplicated (see the exemplary articles in <a href="#near-duplicate-detection">Subsection 2.2</a>).

# <a id="conclusion"></a> Conclusion

I presented POLUSA, a large dataset of online news articles covering policy topics. POLUSA aims to represent the news landscape as perceived by an average US news consumer between Jan. 2017 and Aug. 2019. To achieve this, I performed a series of preprocessing steps, including near-duplicate removal as well as balancing temporally and based on the popularity of outlets. Furthermore, I assigned political leanings to outlets based on prior results. POLUSA enables studying a variety of subjects, e.g., media effects and political partisanship. Due to its size, the dataset allows to utilize data-intense deep learning methods.
