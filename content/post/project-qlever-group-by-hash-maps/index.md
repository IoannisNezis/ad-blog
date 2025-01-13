---
title: "Optimizing GROUP BY in QLever using Hash Maps"
date: 2024-03-08T14:23:59+01:00
author: "Fabian Krause"
authorAvatar: "img/ada.jpg"
tags: ["QLever", "SPARQL", "Hashmap", "Optimization", "GROUP BY"]
categories: []
image: "img/project-qlever-group-by-hash-maps/header.png"
draft: false
---

The current algorithm for evaluation of `GROUP BY` in the SPARQL engine
<a href="https://qlever.cs.uni-freiburg.de/" target="_blank">QLever</a> requires its input to be sorted.
In this project, we improve the performance of `GROUP BY` with the aid of 
hash maps, which allow us to skip sorting the input.

<!--more-->
---
## Content
1. [Introduction](#1-introduction)
2. [Optimization](#2-optimization)
3. [Implementation](#3-implementation)
4. [Benchmarks](#4-benchmarks)
5. [Conclusion](#5-conclusion)
----
## 1. Introduction
`GROUP BY` allows us to reduce certain columns based on aggregate functions, such as `AVG`, `SUM` and `COUNT`.
For example, the following SPARQL query gives us the average number of paper authors per year in the <a href="https://dblp.org/" target="_blank">DBLP</a>
database, uncovering an interesting trend (you can try it out yourself <a href="https://qlever.cs.uni-freiburg.de/dblp/?query=PREFIX+dblp%3A+%3Chttps%3A%2F%2Fdblp.org%2Frdf%2Fschema%23%3E%0ASELECT+%3Fx+%28AVG%28%3Fz%29+AS+%3Favg%29+WHERE+%7B%0A++%3Fy+dblp%3AyearOfPublication+%3Fx+.%0A++%3Fy+dblp%3AnumberOfCreators+%3Fz%0A%7D+GROUP+BY+%3Fx%0A" target="_blank">here</a>): 
```sparql
PREFIX dblp: <https://dblp.org/rdf/schema#>
SELECT ?x (AVG(?z) AS ?avg) WHERE {
  ?y dblp:yearOfPublication ?x .
  ?y dblp:numberOfCreators ?z
} GROUP BY ?x
```
<center style="margin-top:-35px;margin-bottom:55px;">Figure 1: DBLP Query.</center>

In QLever, the current implementation for evaluation of such aggregates relies on the input to the `GROUP BY`
operation to be sorted on the column that is to be grouped by. In the above query, this is not the case:
The result of the join of `?y dblp:yearOfPublication ?x` and `?y dblp:numberOfCreators ?z` on `?y` yields an intermediate
result sorted on `?y`, as QLever uses sort-merge join. Hence, the intermediate result has to be sorted on `?x` before grouping. 
In this project, we investigate an optimization that uses hash maps to optimize this case
by skipping the sorting step.

## 2. Optimization
![Figure x: Pre-Sorted Group By](/img/project-qlever-group-by-hash-maps/ClassicGroupBy.drawio.png)
<center style="margin-top:-35px;margin-bottom:55px;">Figure 2: Operation of Pre-Sorted Group By.</center>

The current implementation of `GROUP BY` (see Figure 2) works as follows: First, the input has to be sorted by the grouped by column. Then, data is iterated through once. Each group is evaluated in a reduce-like fashion, i.e. an accumulator is somehow incremented for each row belonging to a group. Given \\(n\\) rows, the time complexity of sorting is \\(\mathcal{O}(n \log n)\\), producing the final result takes \\(\mathcal{O}(n)\\), yielding an overall complexity of $$\mathcal{O}(n \log n).$$

![Figure x: Hash Map Group By](/img/project-qlever-group-by-hash-maps/HashMapGroupBy.drawio.png)
<center style="margin-top:-35px;margin-bottom:55px;">Figure 3: Operation of Hash Map Group By.</center>

In the proposed optimization (see Figure 3), a map is created, storing an accumulative data structure for each group. 
Afterwards, the keys of this map are sorted, and a result is computed for each group given its stored data. Creation of the map has amortized
complexity of \\(\mathcal{O}(n)\\), as hash map insertions are \\(\mathcal{O}(1)\\) on average, 
sorting the keys \\(\mathcal{O}(m \log m)\\) given \\(m\\) groups, producing the result \\(\mathcal{O}(m)\\), as map access (on average) and calculation of results from
aggregate data structures
are constant, yielding
an overall complexity of $$\mathcal{O}(n + m \log m).$$
Note that sorting the result is not necessary for this optimization, but is due to a constraint specific to QLever,
which expects the result of a `GROUP BY` to be sorted on the grouped columns. Without this constraint, the complexity
becomes \\(\mathcal{O}(n + m)\\).

For \\(m \ll n\\), the optimization should be more time-efficient, as only the small number of groups has to be sorted.
It does, however, use more space. The Pre-Sorted Group By implementation needs only to store the input. As groups
are reduced independently of each other, no additional memory is required for creation of the final result, apart from
a small overhead of keeping an accumulator. For the Hash Map Group By, we need \\(\mathcal{O}(m)\\) to store the map.

In the case that \\(m \approx n\\), sorting the keys of the map can be as expensive as sorting the whole input.
Because of the non-trivial time and space overhead of map creation, the Pre-Sorted Group By implementation should be used in those cases.
Even when not sorting the result, there can be cases where sorting the input is more performant than map insertions.
For example, for very small \\(n\\), or when the underlying array of the map has to be reallocated often due to a repeatedly high load factor, or when there are many collisions, the averaged constant time for map insertion might be larger than \\(\log n\\).


## 3. Implementation
First, a vector of aggregate data is created for each aggregate in the query. Aggregate data stores the information of an aggregate for a group, e.g. a counter for `COUNT`. 
The map itself only stores offsets to these vectors, see Figure 4. This is more efficient than directly storing the vectors in the map since reallocation becomes costly,
especially when dealing with multiple aggregates.

<img src="/img/project-qlever-group-by-hash-maps/MapVectors.drawio.png" style="max-width: 800px;"></img>
<center style="margin-top:-35px;margin-bottom:55px;">Figure 4: Indirect storage of aggregate data.</center>

The algorithm iterates over the data once. For each row, it is checked whether the value in the grouped by column is in the map. If so, we have a vector index to the data structure of the aggregate value of this group. We call the function `addValue` on this data structure, which takes the value of the column in question and, for example, increments the counter in the case of `COUNT`. If the value is not present, another row is added to the vector, and the offset is stored in the map.

Aggregates may contain non-trivial expressions, for example, `AVG(?y + 2)`.
These expressions are evaluated before passing their results into `addValue`.
QLever provides an optimized implementation of expression evaluation on a large number of rows,
hence we evaluate child expressions of aggregates in blocks.

When the whole table has been seen, we sort the keys of the map. Afterwards, we call `computeResult` on each of the group aggregate information data, giving us the result as a `ValueId`. For `COUNT`, this simply creates a numeric `ValueId` from the value of the variable that has been incremented.
Creating the result can be more involved: For `GROUP_CONCAT`, a new entry in the `LocalVocab` has to be created
for the concatenated word.

For expressions that contain aggregates, e.g. `3 * AVG(?y)`, we extend expression evaluation
to support vectors as leafs in the expression tree. We can then
substitute away any occurrences of aggregates with their respective results before
evaluating the root operation, see Figure 5. Expressions that contain the grouped by variable, e.g. `?x + AVG(?y)`, are handled similarly. Again, block-wise expression evaluation is used.
Depth-first search is used to find occurrences of aggregates in the expression tree.

<img src="/img/project-qlever-group-by-hash-maps/Substitution.drawio.png" style="max-width: 800px;"></img>
<center style="margin-top:-35px;margin-bottom:55px;">Figure 5: Evaluation of non-trivial expressions.</center>

## 4. Benchmarks
To evaluate the performance of this optimization, benchmarks were run on
a table of
\\(10^7\\) rows of two columns of type `Double`. Values of the first column were generated
with consecutive group numbers and subsequently shuffled.
The second column was filled completely at random. Each `GROUP BY` operation was
evaluated four times, and the average runtime was taken. 

An important factor that influences the performance of `GROUP BY` is multiplicity, the proportion of rows to distinct rows of a certain column. Multiplicity directly
relates to the number of groups: Given \\(n\\) rows and \\(m\\) groups (i.e. distinct rows of the grouped by columns), 
$$\text{multiplicity} = \frac{n}{m}.$$

Figure 6 shows the measured speedup, i.e. the time required by Pre-Sorted Group By divided by the time required by Hash Map Group By. At low multiplicity, Pre-Sorted Group By performs better, as suggested in our earlier analysis. In cases with a moderate number of groups, we get a considerable speedup of up to 440%.
<img src="/img/project-qlever-group-by-hash-maps/speedup.png" style="max-width: 800px;"></img>
<center style="margin-top:-35px;margin-bottom:55px;">Figure 6: Speedup `t_{"presorted"} // t_{"hashmap"}` for <code>AVG</code>, <code>COUNT</code>,
<code>GROUP_CONCAT</code>, <code>MAX</code>, <code>MIN</code> and <code>SUM</code> for various multiplicities of the grouped by column.</center>

Benchmarks were also run to determine optimal block size for block-wise evaluation of
expressions. These are not included here for brevity's sake. It was found
that block size did not have a huge impact on performance as long as it is large enough.
We settled on the value of 262144.

To investigate the performance impact of the sorting of the results,
benchmarks were run where the results were not sorted. The results can be seen in Figure 7. Hash Map Group By performs
better at all multiplicities, except for operations `MIN`, `MAX` and `GROUP_CONCAT` at a high number of groups.
As suspected, sorting the result is the bottleneck when there are many groups.
<img src="/img/project-qlever-group-by-hash-maps/speedup_no_sorting.png" style="max-width: 800px;"></img>
<center style="margin-top:-35px;margin-bottom:55px;">Figure 7: Speedup `t_{"presorted"} // t_{"hashmap"}` for <code>AVG</code>, <code>COUNT</code>,
<code>GROUP_CONCAT</code>, <code>MAX</code>, <code>MIN</code> and <code>SUM</code> for various multiplicities of the grouped by column, without sorting
the result.</center>

An important factor in reduced performance at low multiplicity is the size of the map.
A larger map leads to more cache misses when accessing the aggregate data,
a problem that Pre-Sorted Group By does not have, as aggregate data is always directly available due
to block-wise evaluation of groups.

## 5. Conclusion
We have implemented an optimization for `GROUP BY` that yields considerable performance improvements of up to 440%.
Overall performance could be further improved by adding a runtime corrector that detects whether the input
seems to have low multiplicity, i.e. the hash map grows very large, and if that is the case switches back to
Pre-Sorted Group By. Additional analysis is required to determine why the speedup of `MIN`, `MAX` and `GROUP_CONCAT`
is comparatively lower than that of the other operations.