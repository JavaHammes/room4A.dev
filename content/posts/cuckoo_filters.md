+++
date = '2025-07-26T16:31:28+02:00'
draft = false
title = 'No Birds Were Harmed in the Making of This Data Structure'
+++

Let's get this out of the way: no, this post isn't about [ornithology](https://en.wikipedia.org/wiki/Ornithology). And yes, I did choose this topic *at least partly* because of the name: **Cuckoo**.

Go ahead, say it out loud: *“Cuckoo.”*

But while the name comes from a famously pushy bird species, what we're really talking about here is a very clever and very efficient **probabilistic data structure**: the **Cuckoo filter**.

Unlike its avian namesake, this filter isn't interested in nests, it's more into fast lookups, low memory usage, and an ability to delete things (a rare treat in this data structure family).

If you've ever heard of a **Bloom filter**, the Cuckoo filter is its more flexible, deletion-friendly cousin. Still fast. Still space-efficient. Just... better at kicking things out when it needs to.

In this post, we'll:
- Demystify what a Cuckoo filter is (no ornithology degree required),
- Compare it to Bloom filters so we know why anyone would bother,
- And build one from scratch in C, because what's more fun than implementing a space-efficient hash structure with a name that makes you giggle?

---

## What are Cuckoo Filters?

### Short Answer:
A Cuckoo filter is a small and fast way to check if something is probably in a set. It uses little memory, works quickly, and lets you remove items, while still keeping errors low.

### The Longer Explanation:

Cuckoo filters are a modern alternative to [Bloom filters](https://systemdesign.one/bloom-filters-explained/) for approximate set membership queries. They allow you to check whether an item is probably in a set, with the guarantee that you'll never get a false negative (i.e., if it says "no", the item is definitely not there).

Unlike Bloom filters, Cuckoo filters support deletions and retain high lookup performance even when nearly full. This makes them ideal for large-scale systems, like caches, databases, or networking equipment, where fast, memory-efficient lookups matter more than perfect accuracy.

### A Quick Detour: Bloom Filters

Before we can appreciate why Cuckoo filters are cool, we've got to take a quick look at what came before them: **Bloom filters**. Think of them as the grandparent in the family of "probably-in-the-set" data structures.

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

That's where Cuckoo filters come in.

### What is a Cuckoo Filter?

So now that we've seen what Bloom filters can (and can't) do, let's meet their smarter, cooler cousin: the **Cuckoo filter**.

Cuckoo filters are based on [Cuckoo hashing](https://link.springer.com/chapter/10.1007/3-540-44676-1_10), which gets its name from the cuckoo bird that lays its eggs in other birds' nests, and, like in the algorithm, if something's in the way, it's kicked out to make room.

Compared to Bloom filters, Cuckoo filters have a few standout features:

- You can delete items.
- They stay fast and reliable even when the filter is almost full.
- They can actually use less space than Bloom filters.
- And bonus: they're easier to implement than some of the other fancy alternatives like [Quotient filters](https://systemdesign.one/quotient-filter-explained/).

#### How It Works

Instead of storing full items, a Cuckoo filter stores short fingerprints, small hash-based representations of each item.

Each fingerprint can go in one of two possible buckets, which are chosen using a pair of hash functions. If both buckets already have something in them?

No problem. The filter just kicks out an existing fingerprint, tries to move it to its other bucket, and keeps repeating that until something lands in an open slot, or until it gives up after a few tries.

Lookups and deletions work the same way: calculate the fingerprint, check both buckets, and see if it's there. If it is, great. If not, it's not in the set.

Because the filter only stores small fingerprints instead of full items, it saves space. And thanks to the clever placement and kicking strategy, it manages to keep performance high even as it fills up.

Let’s keep going with the same locker hallway idea from earlier, but now we're switching from *Bloom filters* to *Cuckoo filters*.

In a Cuckoo filter, instead of flipping bits in lockers, we actually store short fingerprints, tiny codes that represent the items we care about. And each fingerprint is only allowed to live in two possible lockers, chosen by hashing the original item.

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

We'll go into more detail when we implement it.

### In Short

Cuckoo filters are:

- Like Bloom filters, but with deletion and better space efficiency in many real-world cases.
- Based on Cuckoo hashing, which lets them keep performance high even when the structure is almost full.
- Easy to implement, surprisingly powerful, and honestly just kind of fun to say.

*"Cuckoo."*

(Still not over it.)

Check out [Probabilistic Filters by Example](https://bdupras.github.io/filter-tutorial/) for an interactive comparsion of Cuckoo and Bloom filters in action.

## Implementing a Cuckoo filter in C

Now the fun begins.

By now, you should have a general understanding of what a Cuckoo filter is and how it works. Let's move on to implementing one.

The goal here is to implement the core functionality of a Cuckoo filter in C, writing the functions in a way that stays as close as possible to the pseudo code from the [original paper introducing Cuckoo filters](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf):

This way, it's easier to understand how the algorithm works under the hood, and the code will closely reflect the concepts described in the paper.

> Code Repository: The full source code for this Cuckoo filter implementation in C is available on [GitHub](https://github.com/JavaHammes/cuckoo).

1. **Insert**

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

```text
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

1. **Q**: What hash function do we use? **A**: We use [MurmurHash3](https://github.com/PeterScott/murmur3) (x86_32) for both the key and the fingerprint. It’s fast, well‑distributed, and easy to drop in.

2. **Q**: How is the fingerprint calculated?
**A**:
```c
MurmurHash3_x86_32(&key, sizeof(key), FP_HASH_SEED, &raw);
fingerprint = (uint16_t)(raw & 0xFFFF);
```
We simply truncate Murmur3’s 32‑bit output to 16 bits.

3. **Q**: What size should fingerprints be? **A**: Fingerprint length `f` must satisfy `f ≥ log₂(1/ε) + log₂(2b)` where `ε` is your target false‑positive rate and `b` is bucket capacity. For `ε≈0.001` and `b=4`, `16 bits` is plenty.

4. **Q**: What is a bucket? **A**: A bucket holds up to `BUCKET_SIZE` fingerprints (we choose 4). It's just a tiny array.

5. **Q**: How many entries per bucket? **A**: We use 4 entries. The original paper shows that for `ε` between `0.00001` and `0.002`, 4‑entry buckets minimize total space.

6. **Q**: How many buckets do we need? **A**: Pick a power‑of‑two large enough to hold your max items at `~95%` load. In our tests we used `2 048 buckets × 4 slots = 8 192 capacity`;

7. **Q**: What is `MaxNumKicks`? **A**: The maximum number of displacement attempts per insert. We use `500`, as in the paper, it balances insert success rate against latency.

Alright let's get started.

### 1. Breaking the Filter into Four Files

To keep things sane, we'll split our implementation into four logical modules:

1. **Entry** (`cuckoo_entry.[ch]`)
   - Compute a tiny fingerprint (16 bits) with Murmur3
   - Hash the original key -> raw `h1`
   - Hash the fingerprint -> raw `h2 = h1 xor hash(fp)`

2. **Bucket** (`cuckoo_bucket.[ch]`)
   - A fixed‑size array of fingerprints (e.g. 4 slots)
   - `add`, `remove`, and `contains` operations

3. **Table** (`cuckoo_table.[ch]`)
   - An array of buckets (choose a power‑of‑two count, e.g. 1024 or 2048)
   - "Direct" insert (try both buckets without kicking)
   - Lookup and delete by checking both buckets

4. **Filter** (`cuckoo_filter.[ch]`)
   - The high‑level API:
     - `cf_create(bucket_count, max_kicks)`
     - `cf_insert(key)` -> does direct insert -> kicks on full -> up to `max_kicks`
     - `cf_lookup(key)` -> rebuild fingerprint, check both buckets
     - `cf_delete(key)` -> same as lookup but removes on match

And for fun, I also wrote a **`main.c`** demo and a **`test_cuckoo_filter.c`** suite with automated checks (basic inserts, duplicates, overflow, false positives, memory usage, you name it).

### 2. Deep Dive Into The Implementation

Below is our actual `cf_insert`, `cf_lookup`, and `cf_delete`, annotated line‑by‑line.

#### 1. `cf_insert`

```c
cuckoo_status cf_insert(cuckoo_filter *cf, uint32_t key) {
        // 1) Build the entry: fingerprint + two raw hashes
        cuckoo_entry *e = create_cuckoo_entry(key);
        if (!e) {
                return CF_NO_SPACE;
        }

        // 2) Try the "cheap" direct insert into either bucket
        if (table_insert_direct(cf->table, e) == CF_FOUND) {
                destroy_cuckoo_entry(e);
                return CF_FOUND;
        }

        // 3) Kick‑out loop: pick one of the two buckets at random
        uint32_t cur_hash = (rand() & 1) ? e->hash1 : e->hash2;
        cuckoo_fingerprint fp = e->fp;

        for (int n = 0; n < cf->max_kicks; n++) {
                // 3a) Which bucket are we in?
                int idx = cur_hash & (cf->table->bucket_count - 1);
                cuckoo_bucket *b = &cf->table->buckets[idx];

                // 3b) Pick a random victim to evict
                int victim = rand() % b->count;
                cuckoo_fingerprint tmp = b->fingerprints[victim];

                // 3c) Swap: place our fp in the bucket, take the victim’s fp
                b->fingerprints[victim] = fp;
                fp = tmp;

                // 3d) Compute the “alternate” bucket for the evicted fp
                uint32_t hash_alt;
                MurmurHash3_x86_32(&fp.value, sizeof(fp.value), POS2_HASH_SEED,
                                   &hash_alt);
                cur_hash = cur_hash ^ hash_alt;

                // 3e) Try to add the evicted fp into its other bucket
                int alt_idx = cur_hash & (cf->table->bucket_count - 1);
                if (bucket_add(&cf->table->buckets[alt_idx], fp)) {
                        destroy_cuckoo_entry(e);
                        return CF_FOUND;
                }
        }

        // 4) Too many kicks -> give up
        destroy_cuckoo_entry(e);
        return CF_NO_SPACE;
}
```

#### 2. `cf_lookup`

```c
cuckoo_status cf_lookup(cuckoo_filter *cf, uint32_t key) {
        // 1) Build a temporary entry: compute fingerprint + hash1 + hash2
        cuckoo_entry *e = create_cuckoo_entry(key);
        if (!e) {
                return CF_NOT_FOUND;
        }

        // 2) Check the two possible buckets for our fingerprint
        cuckoo_status s = table_lookup(cf->table, e);
        destroy_cuckoo_entry(e);
        return s;
}
```

#### 3. `cf_delete`

```c
cuckoo_status cf_delete(cuckoo_filter *cf, uint32_t key) {
        // 1) Build a temporary entry: compute fingerprint + hash1 + hash2
        cuckoo_entry *e = create_cuckoo_entry(key);
        if (!e) {
                return CF_NOT_FOUND;
        }

        // 2) Try to remove the fingerprint from one of the two buckets
        cuckoo_status s = table_delete(cf->table, e);
        destroy_cuckoo_entry(e);
        return s;
}
```

> See how to run the code on [GitHub](https://github.com/JavaHammes/cuckoo).

All tests passed! Here's how reality lines up with theory:

```
1: === test_basic ===
1: Inserted 5/5 keys
1: Lookup 5/5 present
1: === test_duplicates ===
1: Lookup after 2 inserts: PASS
1: Lookup after 1 delete: PASS
1: Lookup after 2 deletes: PASS
1: === test_delete_nonexistent ===
1: Delete non-existent: PASS
1: === test_overflow ===
1: Inserted until full: 7947 entries
1: Insert after full: FAIL
1: === test_performance ===
1: Inserted 100000 items in 2.309s
1: Looked up 100000 items in 0.011s
1: === test_memory_usage ===
1: Inserted 8192 items; memory used: 24608 bytes (0.02 MB), ~3.00 bytes/item
1: === test_false_positive_rate ===
1: False positives: 3/50000 = 0.0001
1: === test_reinsertion_after_delete ===
1: Lookup after delete: PASS
1: Lookup after reinsertion: PASS
1: === test_random_mix ===
1: Random mix mismatches: 0/10000
1/1 Test #1: CuckooTests ......................   Passed    4.45 sec
```

- Capacity:
    - Observed `~7 947` inserts before failure vs. theoretical max `8 192` (`2 048 buckets × 4 slots`).
    - Slightly lower due to random kicks failing before absolute full.

- Throughput:
    - Inserts: 100 000 in 2.3 s (`~43 K ops/s`)
    - Lookups: 100 000 in 0.011 s (`~9 M ops/s`)

- Memory Usage:
    - `8 192 entries × (2 B fingerprint + 1 B per‑bucket counter) -> 24 608 B total (~3 B/item)`.

- False‑Positive Rate:
    - Measured `3/50 000 = 0.0001 (0.01%)`
    - Very close to theoretical `ε ≤ 2b/2^f ((2 × 4)/2^16) ≈ 0.000122)`

#### 4. Differences to the "Paper" / Real‑World Caveats

Theory and practice don't always match exactly.

1. Hash Function
    - Paper: `CityHash`.
    - Ours: `MurmurHash3_x86_32` for simplicity and portability.

2. Fingerprint Length
    - Paper: `f` is tuned per `ε` and `b`.
    - Ours: Fixed at 16 bits (`uint16_t`)

3. Bucket Size
    - Paper: 2 or 4 entries.
    - Ours: Hard‑coded to 4. You can tweak `BUCKET_SIZE` in `cuckoo_bucket.h`.

4. Table Size
    - Paper: Any capacity ≥ items/max_load.
    - Ours: Power‑of‑two (e.g. 2048) to make mod a bitmask.

5. Resize / Dynamic Scaling
    - We don't support growing or shrinking. Real systems may need a "rehash to bigger table" path once load > 95%.

## Limitations and Considerations

By now, Cuckoo filters might sound like the silver bullet for all your approximate set membership needs. And honestly, they are great, but no data structure gets away without a few trade-offs. So before you go replacing every Bloom filter in your infrastructure with a Cuckoo filter, let's talk about the fine print.

1. No Duplicate Counting
    - Cuckoo filters aren't designed to track how many times an item was inserted. Insert the same key twice, and it'll still only take one slot. If your use case needs frequency tracking, this isn't your tool.
    - Workaround: Consider [counting Cuckoo filters](https://arxiv.org/abs/2003.03801), which embed a small counter alongside each fingerprint. This adds a few extra bits per slot but lets you track insertion counts.

2. Resize Resistance
    - Unlike some structures that support dynamic resizing (e.g. vectors, hash tables), a basic Cuckoo filter doesn't grow once it's full. When insertions start failing, your only option is to reallocate a larger table and reinsert everything.
    - Some advanced implementations support resizing or even use multiple filters chained together, but that adds complexity (and code). For embedded systems or bounded workloads, this isn't a dealbreaker. For dynamic workloads, keep this in mind.

3. Insert May Fail
    - This one surprises people: inserts can fail even when there's technically space left. If the kick-out chain exceeds `MaxNumKicks`, the item isn't added. You're trading simplicity for a small risk of failure. In our tests, insertions succeeded up to about 97% full, but after that, it gets dicey.
    - You could retry with a new fingerprint or even rehash the entire table, but again, that’s complexity you'll have to manage.

4. False Positives Still Exist
    - Like any approximate membership structure, Cuckoo filters can lie. False-positive rate ≤ `2·bucket_size / 2^f`, so you can make FP rates arbitrarily low by increasing fingerprint size, but never zero.
So if your application requires absolute certainty, like a password blacklist or medical diagnosis system... yeah, don't.
    - Also, remember that deleting an item that was never inserted can remove something else if its fingerprint matches, a rare edge case, but one worth noting.

5. Fingerprint Collisions
    - By storing just a fingerprint, we lose the ability to distinguish between different keys that hash to the same tiny value. That's the price we pay for space efficiency. It's rare (especially with 16-bit fingerprints), but it's why deletions must be used carefully and why lookups can lie.

## Conclusion

So, where does that leave us?

Cuckoo filters are a delightful blend of simplicity and power. They're easy to implement, fast in practice, and solve some of the biggest headaches with Bloom filters, like deletions and poor performance at high capacity. For many modern systems, they're a rock-solid choice for approximate membership tests.

If you're building a high-throughput system that needs fast lookups, tight memory usage, and occasional deletes (think: caches, security filters, deduplication systems), Cuckoo filters are a very strong contender. Just keep an eye on their limits, especially if you expect the filter to grow, and be mindful of the rare-but-real quirks of false positives and fingerprint collisions.

And hey, if nothing else, at least now you know there's a data structure out there that shares its name with a bird that evicts its neighbors' kids. Nature is wild.

Thanks for sticking with me through all the fingerprints, buckets, kicks, and C code. I hope this post gave you a new tool for your mental toolbox, and maybe even made you laugh once or twice.

Until next time:

“Cuckoo.”

(Still funny.)

> Source Code: [GitHub](https://github.com/JavaHammes/cuckoo).

PS: I've noticed that every post gets longer and denser. I promise the next one will be short and simple.

---

"It is not the strongest of the species that survives, nor the most intelligent, but the one most responsive to change." - Charles Darwin
