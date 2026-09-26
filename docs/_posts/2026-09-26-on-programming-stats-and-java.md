---
layout: post
title:  "On programming, statistics, and Java"
date:   2026-09-26
categories: programming stats optimizations
---

*With love, to my dear girlfriend*

<small> *poof* </small>

That's the sound of this blog being born into existence, as this is the first post on it. It is also about the problem that gave me the idea to have a blog in the first place.

As the people coming from my Linkedin might know, I am wrapping up my internship at Amazon. It wasn't the usual internship, though, in the sense that I did not have my own self-contained project, but rather I was part of *some "stabilisation initiative"* for an existing project.

<small> *As a side note, I believe that me having to work on production systems has allowed me to truly enjoy this internship, and I hope more internships would take this approach. It also allowed me to see how fun it is to benchmark and see colorful lines go up and down!* </small>

Coming back, I was part of a fairly small team, around 4.5 people (.5 due to 1 being active in 2 projects), so, honestly, work has felt more like a startup than a FAANG (or MAANG, MANGOS, GAYMAN, or however it will be next week). I basically had my cake and ate it too, since I had a rather large amount of ownership and freedom in my tasks.

Today I won't focus on the boring parts (integration tests), but rather I want to tell you, my dear reader, about **how my stats course finally mattered at the job, and about how much I hate Java**.

---

# Context
The service I spent the most of my time working on was a DataStore service. As the name says, it is about... *storing data*.

The initiative's *(I will use this term for lack of a better one; I promise to not use corpo-speak too much 🤙)* goal for this service was **scalability**. How much scalability? How about **being able to take on 1TB ingestions?**

*I am trying to provide as little context as possible so that I don't somehow violate my NDA, but the overarching project was working with regulatory data, and we had to ingest and store this data on our end*

For context, from my measurments (*and previous **refactor and optimisation** of the service*) we were able to ingest **~400 GB\***.

Now, let's start with the interesting stuff!

<small> \* - This is extrapolated. The measurments were taken in the Beta stage, where we had a container of 30GB RAM, and the Prod. containers had 120GB, so in reality I measured 100GB </small>

---

# The problem
Now, the main problem was that we were holding a `HashSet<String>` in memory for a later pass after the ingestions. We basically had something like this:

```
processed = Set()
for ingestion in input_data:

    // do stuff

    processed.add(ingestion.str_that_is_not_PK) // please pay attention to this str_that_is_not_PK!

    // do more stuff, also db.insert(ingestion)

for streamed_row in db.get_rows():

    if streamed_row.str_that_is_not_PK in processed:
        continue

    // do stuff
```

<small> *There might have been a better arch. but I wasn't going to overhaul already existent data, so I did what I could* </small>

As you can probably see, the memory will grow in a *more or less* linear fashion. Saying more or less because there is also GC involved and whatnot.

---

# 1st try at optimizing
*At first (slight spoilers)* I believed that we were inserting the PK into the sets, which happened to be a SHA-256 **string**, meaning it had **64 bytes instead of 32**. I think you can also see where I attacked first!

In my naivite, retrospectively, I created a new class `HashedSHAKey`, which stored just 4 `long` variables. A `long` in Java is 8 bytes, so `8 x 4 = 32`, 50% out of 64. 

**Nice! We probably saved ~50% of the memory! We can pat ourselves on the back and call it a day!** We could now scale to ~800GB, probably way above this project's needs in this lifetime. Let's just see what the tests say...

*-18% in memory utilization...*

**What the f\*#k??** This does not make any sense. Let me prompt Claude to see what it can *hallucinate* <small> (and verify the claims myself of course) </small>.

*It seems that for every object in Java, an additional 12-16 bytes are allocated as the header of the object*. So for every `HashedSHAKey`, instead of 32 bytes, I had like 48. *That meant that I was using 50% more memory than I anticipated, meaning my upper bound was 25\% overall improvement*. With whatever schenenigains Java does under the hood for HashSet (*which, btw, IS significant for the memory overhead*), the math started mathing.

---

# Libraries & probabilities
I moved on to a different approach, and eyed this one library *with close to 0 overhead*, **fastutil**. There was one problem though: **there aren't any 32-byte HashSet implementations, only 8-byte**.

Now, narrowing *what I thought to be* full SHA-256 strings to 8 bytes seemed **risky, to say the least**. Thankfully, my mentor, who had a master in statistics, thought the same thing and prompted me to compute the *probabilities of false-positives in the narrowing case*.

*This started a series of 3 days of a vicious cycle*
```text
               ┌────────────────────┐
               │                    │
               ▼                    │
┌─────────────────────────────┐     │
│  1. Computed something      │     │
└──────────────┬──────────────┘     │
               ▼                    │
┌─────────────────────────────┐     │
│  2. Got happy — result      │     │
│     seemed good             │     │
└──────────────┬──────────────┘     │
               ▼                    │
┌─────────────────────────────┐     │
│  3. Girlfriend doubted my   │     │
│     statistical competence  │     │
└──────────────┬──────────────┘     │
               ▼                    │
┌─────────────────────────────┐     │
│  4. Started double-guessing │     │
│     myself                  │     │
└──────────────┬──────────────┘     │
               ▼                    │
┌─────────────────────────────┐     │
│  5. Debated with the        │     │
│     hallucination machines  │     │
└──────────────┬──────────────┘     │
               │                    │
               └────────────────────┘
                    (loop forever)
```

Ultimately I got sick of wondering whether to use the birthday formula *(my mentor put that thought in my head; didn't even know what it was used for before)* or not, so I took my own approach.

**You can skip the math here if you'd like, but it is interesting!**

We are also taking the worst case scenario, in which we constantly add rows to our database.


```
Goal = 1TB
Avg. row size = 2KB

N = 500M
Precision = 64 (so 2^64 total possibilities)

M = # of iterations (the # will grow each run by N) = 0, 500M, 1B, 1.5B, ... = i * N

P(false-positive in 1 iteration) = 500M / 2^64 = q
P(no false-positive in 1 iteration) = 1 - q
P(no false-positive in 1 run of M_i iterations) = (1 - q) ^ M_i

P(no false-positives in all R runs) 
    = (1 - q)^0 * (1 - q)^500M * ... 
    = Product_i=0^R [ (1 - q)^i * N ]
    = (1 - q)^(N * (0 + 2 + ... + R - 1))
    = (1 - q)^(N * R * (R - 1) / 2)

```

Let's say that an acceptable rate of false-positive is 50%:


```
0.5 = (1 - q)^(N * R * (R - 1) / 2)

<=>

ln(0.5) = N * R * (R - 1) / 2 * ln(1 - q)

<=>

2 * ln(0.5) / (N * ln(1 - q)) = R * (R - 1)
```

Since q is really small, we can approximate `ln(1 - q) = -q`


```
2 * ln(0.5) / (N * -q) = R^2 - R
```

Now we can replace N and q, solve for R and get...
```
R = 10 (approx)
```

Wow, all this math only to find that, with the naive approach of having 1 set storing the first 8 bytes of the sha256, we'd toss a coin on the 10th run already. **This is bad**.

Hmm, what if *instead of 1 set, we have **2**?* Doing the math again... *(truth be told I used a matlab script for this):*

```
P(no false-positives in both sets) = P(no false-positives in the first set) * P(no false-positives in the second set)
```
<sub> We will suppose we have independent probabilities for simplicity (and I hope it is the case as well in the real world...) </sub>

{% highlight matlab %}

% Basically we have that the prob of an iteration having 0 false-positives
% is (1 - q)

% a run has N iterations, so a run with 0 false-positives has basically
% (1 - q)^N probs of having 0 false-positives

% Then we have to solve Prod_0^R[ (1-q)^N ] >= 0.5, so we find out the 
% R for which the probability of having 0 false-positives goes to less than 50%
N = 500000000;
q = (N / 2^64)^2;

x = 2 * log(0.5) / (N * log1p(-q));

R = ceil((1 + sqrt(1 + 4*x)) / 2);

fprintf('Smallest run = %d\n', R);

{% endhighlight %}

Plugging this in gives us:
```
Smallest run = 1942642
```

Honestly, if the math is right, and by absurd a 1TB ingestion would run each day, **it vastly outlives this project**.

Great! Time to run the test again aaand... Ewreka! **We got the expected 75% reduction in memory usage <small> *(excluding the first 2.3GB the container allocated)* </small>**!

Truth is though, when running a 400GB test on a single container (remember, prod is 4x the beta containers), I somehow got **a ~10x reduction in memory** and I have absolutely no idea why. What Claude hallucinated is that it could be because of some GC shenenigains, but no idea. But nor do I care, since the result is better than what I've hoped for.

<img src="/res/math_paper.jpeg" alt="the 'napkin' math" width="50%">
<small>Ugh, got the wrong approximation formula for ln(1-q) on the pic. Bummer...</small>

---

# Final sanity check
Before checking everything off and putting the PR for reviewing, I decided to trust my gut and look again inside the code. *And yeah, remember when I said str_that_is_not_PK was a 64-character SHA-256 string?* **It wasn't.** It could really be anything; I was confusing it with the PK for the past 3 days.

Now, were my efforts in vain? **Not at all; with the set approach I was hashing the string to sha256 either way, so I was still working on the same probability space**.

Phew, that scared me for a bit. Time to do this PR!

---

# Conclusions
I believe you can see why one might come to hate Java when it comes to certain things. There is no denying it is a good language for *some* things, but it is **certainly not a good language if you have lots of small objects**.

But aside from this, there is a really positive message behind all of this: **math *can* matter**. Yeah, sure, on a daily basis there are very few developers who are going to really need maths beyond simple computations. Now even less with the advent of LLMs, probabily.

But man, oh man! How good it feels to solve problems again! I can say that for the past few days I've had the same feeling I had while programming pre-LLM era. *It felt GREAT*. I truly hope to be have more such opportunities from now on!

But I believe that's all for this post. Hopefully I did not bore you too much, dear reader. 

'Til we see again!

