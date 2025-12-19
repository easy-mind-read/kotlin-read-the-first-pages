---
layout: default
title: "Is Your Clean Kotlin Code Secretly Slow? One Common Mistake"
description: "A deep dive into a common Kotlin performance trap that looks like clean code. Learn to spot and fix hidden work in your collection pipelines."
permalink: /kotlin-slow-code-mistake/
keywords:
  - Kotlin
  - Kotlin performance
  - Kotlin best practices
  - code optimization
  - software mental models
---

# Is Your "Clean" Kotlin Code Secretly Slow?

You write clean, readable, and fluent Kotlin. You use collection functions like `filter`, `map`, and `take`. But what if one of the most common patterns is quietly destroying your application's performance?

This isn't about micro-optimizations. It's about a fundamental assumption that leads developers to write code that is dramatically slower than it needs to be.

---

## The TLDR: Stop Hiding Expensive Work in "Simple" Chains

> A tiny change in **where** you place work (e.g., inside vs. outside a filter) can change runtime behavior from fast and efficient to sluggish and wasteful.

Kotlin's fluent API makes it easy to chain operations. It also makes it easy to hide expensive computations inside those chains, where they get executed far more often than necessary.

---

## The Performance Bug That Looks Like Clean Code

Here’s a mistake I see all the time. At first glance, it looks perfectly fine.

We have a list of items and we want to find the first 10 that match some criteria. The criteria check happens to be an expensive operation.

### Can you spot the problem?

```kotlin
import kotlin.system.measureTimeMillis

// A function that simulates a heavy computation
private fun expensiveScore(s: String): Int {
    var acc = 0
    repeat(50_000) { i ->
        acc = (acc + s.hashCode() + i) xor (acc shl 1)
    }
    return acc
}

fun main() {
    val items = List(2_000) { "item-$it" }

    // The "slow" way:
    val slowMs = measureTimeMillis {
        val result = items
            .filter { expensiveScore(it) % 2 == 0 } // expensiveScore() runs for every item
            .take(10)
        println("Slow result size = ${result.size}")
    }

    // The "fast" way:
    val fastMs = measureTimeMillis {
        val result = items
            .asSequence() // <-- The magic starts here
            .map { it to expensiveScore(it) } // Still runs, but...
            .filter { (_, score) -> score % 2 == 0 }
            .map { (item, _) -> item }
            .take(10) // The chain stops as soon as 10 items are found
            .toList() // Execute the lazy operations
        println("Fast result size = ${result.size}")
    }

    println("slowMs=$slowMs, fastMs=$fastMs")
}
```

### The Explanation

In the "slow" version, `.filter` runs the `expensiveScore()` function on **every single item** in the list of 2,000, just to find the first 10 matches. It does all the work upfront.

In the "fast" version, we introduce `.asSequence()`. This makes the collection operations **lazy**.

1.  **`asSequence()`**: Converts the list into a `Sequence`, which processes items one by one, on demand.
2.  **`take(10)`**: This is the crucial part. It tells the sequence, "I only need 10 items that match the criteria."
3.  **`toList()`**: This is the terminal operation that kicks everything off.

The sequence will now pull items one by one through the `map` and `filter` chain until exactly 10 items have been collected. Instead of running `expensiveScore()` 2,000 times, it might only run it 15 or 20 times (or however many it takes to find 10 even scores). The difference is massive.

---

## Reader Challenge

If `expensiveScore()` were a network call to an API with a rate limit:

1.  Which version of the code would get your IP address blocked?
2.  How would you explain the fix to a junior developer in one sentence?

---

## Get The Full Guide

This post is a sample from a chapter in **Kotlin Read the First, Easy Mind**. If you enjoy this practical, no-fluff approach to learning, you can get the full book here:

- **Amazon**: [Get PreView](https://www.amazon.com/dp/B0G8H4QX2L/)

---

## What to read next
- Go back to the [home page](https://easy-mind-read.github.io/kotlin-read-the-first-pages/)

