---
layout: post
title: "Appreciating Algorithm Complexity"
date: 2013-10-05 12:52
comments: true
categories: 
---

[Algorithms: Design and Analysis, Part 1](https://www.coursera.org/course/algo) is a free online course from Standford offered through [Coursera.org](https://www.coursera.org/). An assignment that stuck with me was implementing an algorithm for counting inversions. Counting inversions is a way to quantify similarity of two ordered lists. The canonical example described in the lectures was comparing lists of favorite movies.

I banged out a naive implementation using nested loops that I could use for comparison to make sure I was implementing the real algorithm correctly.

``` java
private long countQuadratic(int[] a) {
    long count = 0;
    for (int i = 0; i < a.length; i++)
        for (int j = i; j < a.length; j++)
            if (a[i] > a[j])
                count++;
    return count;
}
```
The algorithm described in lecture is a divide and conquer algorithm based on a variation of [merge sort](http://en.wikipedia.org/wiki/Merge_sort), and as you might expect from its lineage, runs in `O(nlog(n))` time.

``` java
private long countLinearithmic(int[] a) {

    // REDACTED

    long countLeft = countLinearithmic(left);
    long countRight = countLinearithmic(right);
    long countSplit = 0;

    int i = 0;
    int j = 0;
    for (int k = 0; k < a.length; k++) {

        // REDACTED

    }

    return countLeft + countRight + countSplit;
}
```

_\* The guts of the implementation are not shown per the [Coursera honor code](https://www.coursera.org/about/honorcode)._

The assignment was to run this algorithm on a list of 100,000 integers and report the number of inversions. I ran both implementations as a sanity check.

``` java
@Test
public void should_count_inversions_in_text_file() {

    int[] a = FileUtil.parseInts("Inversions.txt");
    assertThat(a.length, is(100000));

    long start1 = System.currentTimeMillis();
    long count1 = countQuadratic(a); // O(n^2)
    long duration1 = start1 - System.currentTimeMillis();

    println("countQuadratic: " + duration1);

    long start2 = System.currentTimeMillis();
    long count2 = countLinearithmic(a); // O(nlog(n))
    long duration2 = start2 - System.currentTimeMillis();

    println("countLinearithmic: " + duration2);

    assertThat(count1, is(equalTo(count2)));
}
```

 It's one thing to crunch numbers on a calulator or plot graphs to gain an intuition for algorithm complexity, but it's quite a bit more meaningful to wait for 15 seconds while a 2.7GHz quad-core Intel Core i7 grinds through ~5 Billion comparisions `O(n^2)` versus a mere 140 ms to zip through 1.6 Million comparisons using an `O(nlog(n))` implementation.

 > Yeah, science!

<iframe width="560" height="315" src="//www.youtube.com/embed/eQR1r1KTjaE" frameborder="0" allowfullscreen></iframe>