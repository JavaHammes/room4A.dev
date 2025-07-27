+++
date = '2025-07-26T16:31:28+02:00'
draft = true
title = 'Only a Cuckoo Would Check the Cache Every Time'
+++

Yes, I'll admit it, part of the reason I chose to write about this topic is because of the name: Cuckoo. Just say it out loud with me: *"Cuckoo."* One more time? **"Cuckoo"**. Yeah, that's fun.

No, this post isn't about the bird (though that's where the data structure gets its name), it's about **Cuckoo Filters**.

But what are Cuckoo filters?

Here's the official definition from [Redis](https://redis.io/docs/latest/develop/data-types/probabilistic/cuckoo-filter/):

> A Cuckoo filter, just like a Bloom filter, is a probabilistic data structure that enables you to check if an element is present in a set in a very fast and space-efficient way, while also allowing for deletions and showing better performance than Bloom in some scenarios.

Okay, "probabilistic data structure", "space-efficient", "Bloom filter"... what does any of that even mean?

(If you don't know what a data structure is, this might not be the post for you. But if you're just fuzzy on the rest, no worries, you're in the right place.)

In this post, we'll uncover what's behind the quirky name "Cuckoo" and explore where cuckoo filters shine. We'll also dive into a hands-on implementation in C, and put it to use in a practical scenario: **using a Cuckoo Filter to avoid unnecessary cache lookups.**

---

## Cache Admission Filter: Avoiding Pointless Cache Lookups

### The Basics

- **What's a cache?**
  A cache is a fast-access storage layer that keeps frequently used data close by, so you don't always have to fetch it from a slower backend (like a database or external service).

- **The issue:**
  Even when a key isn't in the cache, systems often still check it, just in case. Each of those checks has a cost: a network round-trip, added latency, or resource usage on the cache server.

### The Idea

Use a **Cuckoo filter** as a quick pre-check before hitting the cache.

- If the filter says **"no"**, the key definitely isn’t in the cache, skip it and go straight to the backend.
- If the filter says **"maybe"**, check the cache as usual.

This avoids unnecessary cache lookups when there's no realistic chance of a hit.

### How It Works

1. **Cuckoo Filter Tracks Cached Keys**
   It stores lightweight fingerprints of the keys currently in cache.

2. **Update on Cache Writes**
   When a new key-value pair is added to the cache, the key's fingerprint is also added to the filter.

3. **Remove on Eviction or Invalidation**
   When a key is evicted from cache (or explicitly removed), its fingerprint is deleted from the filter.

4. **Check on Read**
   - **Not in filter** -> Skip the cache and go directly to the source.
   - **Maybe in filter** -> Check the cache as usual. If it's a miss, fallback to the backend and refresh both the cache and the filter.

This isn't a magic bullet, but if your workload includes lots of failed cache lookups, it's a cheap way to reduce overhead.

## What are Cuckoo Filters?

### Short Answer:
A Cuckoo Filter is a small and fast way to check if something is probably in a set. It uses little memory, works quickly, and lets you remove items, while still keeping errors low.

### The Longer Explanation:

Cuckoo Filters are a modern alternative to Bloom Filters for approximate set membership queries. They allow you to check whether an item is probably in a set, with the guarantee that you'll never get a false negative (i.e., if it says "no", the item is definitely not there).

Unlike Bloom Filters, Cuckoo Filters support deletions and retain high lookup performance even when nearly full. This makes them ideal for large-scale systems, like caches, databases, or networking equipment, where fast, memory-efficient lookups matter more than perfect accuracy.

### A Quick Detour: Bloom Filters

Before we can appreciate why cuckoo filters are cool, we've got to take a quick look at what came before them: **Bloom filters**. Think of them as the grandparent in the family of "probably-in-the-set" data structures.

So what do they do? A Bloom filter lets you ask: "Is this item in my set?" And it'll give you one of two answers:

1. "Nope, definitely not."
2. "Maybe! (but I could be lying)"

That second answer is what makes it probabilistic, you might get a false positive, but never a false negative. That means if it says something isn't in the set, you can trust it. If it says it is... well, it might be wrong.

#### How it works, in a nutshell:

- You hash your item a few times.
- You flip some bits in a large bit-array.
- When you want to check for that item later, you hash it again and check those bits.
- If they’re all still flipped, it *might* be in the set.

Imagine a row of lockers, say, 1,000 lockers in a hallway. All of them are initially closed and locked (representing `0` bits).

Now you want to add the word "banana" to your Bloom filter:
1. You run "banana" through 3 different hash functions.
2. Each hash tells you to go open locker `12`, `348`, and `709`.
3. You go to those lockers and flip them open (set those bits to `1`).

Now someone later asks, “Was ‘banana’ here?”

1. You hash it again and get lockers `12`, `348`, and `709`.
2. If all three lockers are open: *maybe!* There's a chance something else opened those same lockers.
3. If even one is closed: *nope, definitely not.* "Banana" was never added.

#### But Bloom filters have issues:

- You can't delete items (not without extra overhead).
- They slow down and get less accurate when they're close to full.
- Even with good tuning, Bloom filters still use about `44%` more space than the theoretical minimum needed for their accuracy.

Still, for a long time, Bloom filters were the go-to tool for fast, compact set membership checks. But they've got their limits, especially if your use case includes deleting stuff or operating close to full capacity.

That's where cuckoo filters come in.

### What is a Cuckoo Filter?

So now that we've seen what Bloom filters can (and can't) do, let's meet their smarter, cooler cousin: the **Cuckoo Filter**.

Cuckoo filters are based on [Cuckoo hashing](https://link.springer.com/chapter/10.1007/3-540-44676-1_10), which gets its name from the cuckoo bird that lays its eggs in other birds' nests, and, like in the algorithm, if something's in the way, it's kicked out to make room.

Compared to Bloom filters, Cuckoo filters have a few standout features:

- You can delete items.
- They stay fast and reliable even when the filter is almost full.
- They can actually use less space than Bloom filters.
- And bonus: they're easier to implement than some of the other fancy alternatives like [Quotient filters](https://systemdesign.one/quotient-filter-explained/).

#### How It Works

Instead of storing full items, a cuckoo filter stores short fingerprints, small hash-based representations of each item.

Each fingerprint can go in one of two possible buckets, which are chosen using a pair of hash functions. If both buckets already have something in them?

No problem. The filter just kicks out an existing fingerprint, tries to move it to its other bucket, and keeps repeating that until something lands in an open slot, or until it gives up after a few tries.

Lookups and deletions work the same way: calculate the fingerprint, check both buckets, and see if it's there. If it is, great. If not, it's not in the set.

Because the filter only stores small fingerprints instead of full items, it saves space. And thanks to the clever placement and kicking strategy, it manages to keep performance high even as it fills up.

Let’s keep going with the same locker hallway idea from earlier, but now we're switching from *Bloom filters* to *Cuckoo filters*.

In a cuckoo filter, instead of flipping bits in lockers, we actually store short fingerprints, tiny codes that represent the items we care about. And each fingerprint is only allowed to live in two possible lockers, chosen by hashing the original item.

**Same example: adding "banana"**

**Insert:**

1. You hash "banana" to get a fingerprint, let's say it's `7a`.
2. Two hash functions tell you that this fingerprint can live in locker `5` or `12`.
3. You check both:
    - If either locker has space, you store `7a` there.
    - But if both lockers are full?
        - You pick one (say, locker `12`), kick out a fingerprint that's already in there, let's say `3f`.
        - Now you try to move `3f` to its other allowed locker.
        - If that's also full? You kick again and repeat.
        - This continues until someone finds an open locker, or the filter gives up after too many tries.

**Lookup:**

- Hash "banana" again, get lockers `5` and `12`, and check for `7a`. If it's in either one, we return "probably yes". If not, it's definitely not in the set.

**Delete:**

- Same idea, if `7a` is in one of those lockers, just remove it.

Obviously, there’s more going on behind the scenes, but this should be enough to get the general idea across without overcomplicating it.

We'll go into more detail when we implement it (because, let's be honest, we kind of have to).

### In Short

Cuckoo filters are:

- Like Bloom filters, but with deletion and better space efficiency in many real-world cases.
- Based on cuckoo hashing, which lets them keep performance high even when the structure is almost full.
- Easy to implement, surprisingly powerful, and honestly just kind of fun to say.

*"Cuckoo."*

(Still not over it.)

Check out [Probabilistic Filters by Example](https://bdupras.github.io/filter-tutorial/) for an interactive comparsion of Cuckoo and Bloom filters in action.

## Implementing a Cuckoo Filter in C

Now the fun begins.

By now, you should have a general understanding of what a Cuckoo filter is and how it works. Let's move on to implementing one.

The goal here is to implement the core functionality of a Cuckoo filter in C, writing the functions in a way that stays as close as possible to the pseudo code from the [original paper introducing Cuckoo filters](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf):

This way, it's easier to understand how the algorithm works under the hood, and the code will closely reflect the concepts described in the paper.

2. **Insert**

```
// Generate a short fingerprint from the input
f = fingerprint(x);

// Compute two candidate bucket indices
i1 = hash(x);
i2 = i1 xor hash(f);

// Try to insert into either bucket directly
if bucket[i1] or bucket[i2] has an empty entry then
    add f to that bucket;
    return Done;

// If both are full, we need to reloacte items
i = randomly pick i1 or i2;
for n = 0; n < MaxNumKicks; n++ do
    randomly select an entry e from bucket[i];
    swap f and the fingerprint stored in entry e;
    i = i xor hash(f); // compute alternate index for new f
    if bucket[i] has an empty entry then
        add f to bucket[i];
        return Done;

// Too many kicks means the table is too full
return Failure;
```

2. **Lookup**

```
// Recreate the fingerprint and candidate buckets
f = fingerprint(x);
i1 = hash(x);
i2 = i1 xor hash(f);

// If either bucket contains the fingerprint, it might be a match
if bucket[i1] or bucket[i2] has f then
    return True;
return False;
```

3. **Delete**

```
// Recreate the fingerprint and candidate buckets
f = fingerprint(x);
i1 = hash(x);
i2 = i1 xor hash(f);
if bucket[i1] or bucket[i2] has f then
    remove a copy of f from this bucket;
    return True;
return False;
```

From reading the pseudo code a few questions remain. These will be answered in a Q&A in the following:

1. Q: What hash function is used? A: The paper does not mandate a specific hash function. However, in their implementation, the authors use CityHash for its speed and distribution properties.
2. Q: How is the fingerprint calculated? A: `fingerprint(x) = truncate_bits(Hash(x),f)`, with f being the fingerprint length in bits
3. Q: What size should fingerprints be? A: The fingerprint size f depends on both the desired false positive rate (`ε`) and the bucket size (`b`). It must satisfy: `f ≥ log₂(1/ε) + log₂(2b)`
4. Q: What is a bucket? A: A bucket is a container that holds a small number of fingerprints, usually 2 to 4.
5. Q: How many entries does a bucket have? A: The number of entries per bucket is chosen based on the desired false positive rate. For a false positive rate between `0.00001` and `0.002`, a bucket size of 4 entries minimizes space usage. The paper primarily focuses on 2 and 4-entry buckets.
6. Q: How many buckets do we need? A:
7. Q: What is MaxNumKicks? A: MaxNumKicks is the maximum number of displacement attempts during insertion before declaring a failure. The authors use 500 in their implementation, which balances performance and success rate.

Alright let's get started.


## Applying It: Avoiding Unnecessary Cache Lookups


## Limitations and Considerations


## Conclusion

