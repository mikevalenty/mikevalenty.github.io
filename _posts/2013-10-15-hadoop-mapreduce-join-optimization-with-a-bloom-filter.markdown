---
title: "Hadoop MapReduce Join Optimization With a Bloom Filter"
date: 2013-10-15 22:49:00 -0700
layout: post
categories:
---

Doing a join in hadoop with Java is painful. A one-liner in [Pig Latin](http://pig.apache.org/) can easily explode into hundreds of lines of Java. However, the additional control in Java can yield significant performance gains and simplify complex logic that is difficult to express in Pig Latin.

In my case, the left side of the join contained about 100K records while the right side was closer to 1B. Emitting all join keys from the mapper means that all 1B records from the right side of the join are shuffled, sorted and sent to a reducer. The reducer then ends up discarding most of join keys that don’t match the left side.

Any best practices guide will tell you to push more work into the mapper. In the case of a join, that means dropping records in the mapper that will end up getting dropped by the reducer anyway. In order to do that, the mapper needs to know if a particular join key exists on the left hand side.

An easy way to accomplish this is to put the smaller dataset into the `DistributedCache` and then load all the join keys into a `HashSet` that the mapper can do a lookup against.

```java
for (FileStatus status : fileSystem.globStatus(theSmallSideOfTheJoin)) {
    DistributedCache.addCacheFile(status.getPath().toUri(), job.getConfiguration());
}
```

```java
@Override
protected void setup(Mapper.Context context) {
    buildJoinKeyHashMap(context);
}

@Override
protected void map(LongWritable key, Text value, Context context) {

  ...

    if (!joinKeys.contains(joinKey))
      return;

    ...

    context.write(outKey, outVal);
}
```

This totally works, but consumes enough memory that I was occassionally getting `java.lang.OutOfMemoryError: Java heap space` from the mappers. Enter the [Bloom filter](http://en.wikipedia.org/wiki/Bloom_filter).

> A Bloom filter is a space-efficient probabilistic data structure that is used to test whether an element is a member of a set. False positive matches are possible, but false negatives are not. Elements can be added to the set, but not removed. The more elements that are added to the set, the larger the probability of false positives. -Wikipedia

I hadn’t heard of a Bloom filter before taking [Algorithms: Design and Analysis, Part 1](https://www.coursera.org/course/algo). If not for the course, I’m pretty sure I would have skimmed over the innocuous reference while pilfering around the hadoop [documentation](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/util/bloom/BloomFilter.html). Fortunately, recent exposure made the term jump out at me and I quickly recognized it was exactly what I was looking for.

When I took the course, I thought the Bloom filter was an interesting idea that I wasn’t likely to use anytime soon because I haven’t needed one yet and I’ve been programming professionally for more than a few years. But you don’t know what you don’t know, right? It’s like thinking about buying a car you didn’t notice before and now seeing it everywhere.

### Configuration

The documentation is thin, with little more than variable names to glean meaning from.

```java
public BloomFilter(int vectorSize,
                   int nbHash,
                   int hashType)
```

*   `vectorSize` - The vector size of this filter.
*   `nbHash` - The number of hash function to consider.
*   `hashType` - type of the hashing function (see Hash).

I know what you’re thinking. What could be more helpful than *The vector size of this filter* as a description for `vectorSize`? Well, the basic idea is there’s a trade-off between space, speed and probability of a false positive. Here’s how I think about it:

*   `vectorSize` - The amount of memory used to store hash keys. Larger values are less likey to yield false positives. If the value is too large, you might as well use a `HashSet`.
*   `nbHash` - The number of times to hash the key. Larger numbers are less likely to yeild false positives at the expense of additional computation effort. Expect deminishing returns on larger values.
*   `hashType` - type of the hashing function (see Hash). The Hash documentation was reasonable so I’m not going to add anything.

I used trial and error to figure out numbers that were good for my constraints.

```java
public class BloomFilterTests {
    private static BloomFilter bloomFilter;
    private static String knownKey = newGuid();
    private static int numberOfKeys = 500000;

    @BeforeClass
    public static void before() {
        bloomFilter = new BloomFilter(numberOfKeys * 20, 8, Hash.MURMUR_HASH);
        bloomFilter.add(newKey(knownKey));

        for (int i = 0; i < numberOfKeys; i++)
            bloomFilter.add(newKey(newGuid()));
    }

    @Test
    public void should_contain_known_key() {
        assertThat(hasKey(knownKey), is(true));
    }

    @Test
    public void false_positive_probability_should_be_low() {

        int count = 0;
        for (int i = 0; i < numberOfKeys; i++)
            if (hasKey(newGuid()))
                count++;

        int onePercent = (int) (numberOfKeys * .01);

        assertThat(count, is(lessThan(onePercent)));
    }

    private static String newGuid() {
        return UUID.randomUUID().toString();
    }

    private static Key newKey(String key) {
        return new Key(key.getBytes());
    }

    private boolean hasKey(String key) {
        return bloomFilter.membershipTest(newKey(key));
    }
}
```

When you have your numbers worked out, simply swap out the `HashMap` with the `BloomFilter` and then blog about it.
