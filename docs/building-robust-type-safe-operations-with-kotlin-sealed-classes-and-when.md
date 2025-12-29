---
layout: default
title: "Building Robust, Type-Safe Operations with Kotlin Sealed Classes and `when`"
description: "Tired of complex `if/else` ladders and runtime errors in your core logic? Discover how Kotlin's `sealed class` and `when` expression empower you to create highly robust and maintainable systems with explicit error handling."
permalink: /building-robust-type-safe-operations-with-kotlin-sealed-classes-and-when/
---

TL;DR: This post demonstrates how Kotlin's sealed classes, `when` expressions, and extension functions enable building highly robust, type-safe, and maintainable systems for complex operations and explicit error handling.

## The Problem: Fragile Operational Logic and Implicit Error Handling

In many applications, especially those dealing with critical business logic like financial calculations or data transformations, you often encounter situations where you need to perform a variety of distinct operations. Traditionally, this might involve:

1.  **Enum-based operations with limited data:** Using simple enums (`enum class`) to represent operation types, then needing additional parameters that are loosely coupled.
2.  **Verbose `if/else` or `switch` statements:** Handling each operation type often leads to long, error-prone conditional blocks that are hard to maintain and extend.
3.  **Implicit error handling:** Relying on exceptions or nullable return types, which can lead to unhandled errors at runtime or forces developers to constantly check for `null`.
4.  **Lack of compile-time guarantees:** The compiler doesn't always know if you've handled *all* possible operations or error states, pushing potential bugs to runtime.

These issues lead to brittle code, increased debugging time, and a higher risk of production issues.

## The Real-World Scenario: Designing a Financial Modeling Engine

Imagine you're developing a sophisticated financial modeling engine. This engine needs to perform a wide array of calculations: basic arithmetic for portfolio adjustments, percentage calculations for tax or profit margins, power functions for compounding interest, and square roots for volatility analysis. Crucially, every calculation must be precise, auditable, and handle edge cases (like division by zero or square roots of negative numbers) gracefully and explicitly.

Let's see how Kotlin's powerful features provide an elegant solution.

## The Solution: Kotlin's Sealed Classes, `when`, and Explicit Results

Our financial modeling engine requires a robust way to define operations and their parameters, execute them, and communicate results (or errors) clearly.

### Defining Operations with Sealed Classes

First, we define all possible operations using a `sealed class`. This is a powerful feature that allows us to define a restricted class hierarchy, meaning all subclasses must be defined within the same file or module. This restriction is key for type safety and exhaustive checks.

```kotlin
sealed class Operation {
    data class Add(val a: Double, val b: Double) : Operation()
    data class Subtract(val a: Double, val b: Double) : Operation()
    data class Multiply(val a: Double, val b: Double) : Operation()
    data class Divide(val a: Double, val b: Double) : Operation()
    data class Power(val base: Double, val exponent: Double) : Operation()
    data class SquareRoot(val value: Double) : Operation()
    data class Percentage(val value: Double, val percent: Double) : Operation()
}
```

Each operation is a `data class` inheriting from `Operation`. This allows us to encapsulate all necessary parameters directly within the operation itself. For example, `Add` needs two `Double` values, while `SquareRoot` only needs one. This structure makes our operations highly descriptive and type-safe.

### Handling Results with Explicit States

Next, we address the critical aspect of returning results, including errors. Instead of throwing exceptions or returning `null`, we use another `sealed class` to represent the outcome of a calculation:

```kotlin
sealed class CalculatorResult {
    data class Success(val value: Double) : CalculatorResult()
    data class Error(val message: String) : CalculatorResult()
}
```

Now, every calculation *must* return either a `Success` with the calculated `value` or an `Error` with a descriptive `message`. This forces the calling code to explicitly handle both outcomes, significantly improving robustness and preventing unforeseen runtime failures.

### Executing Operations with Exhaustive `when` Expressions

The core logic resides in the `calculate` function within our `Calculator` class:

```kotlin
class Calculator {
    private val history = mutableListOf<String>()

    fun calculate(operation: Operation): CalculatorResult {
        val result = when (operation) { // The magic of 'when' with sealed classes
            is Operation.Add -> {
                val sum = operation.a + operation.b
                history.add("${operation.a} + ${operation.b} = $sum")
                CalculatorResult.Success(sum)
            }
            is Operation.Divide -> {
                if (operation.b == 0.0) {
                    history.add("${operation.a} ÷ ${operation.b} = ERROR (Division by zero)")
                    CalculatorResult.Error("Cannot divide by zero")
                } else {
                    val quotient = operation.a / operation.b
                    history.add("${operation.a} ÷ ${operation.b} = $quotient")
                    CalculatorResult.Success(quotient)
                }
            }
            // ... other operations ...
            is Operation.SquareRoot -> {
                if (operation.value < 0) {
                    history.add("√${operation.value} = ERROR (Negative number)")
                    CalculatorResult.Error("Cannot calculate square root of negative number")
                } else {
                    val result = Math.sqrt(operation.value)
                    history.add("√${operation.value} = $result")
                    CalculatorResult.Success(result)
                }
            }
        }
        return result
    }

    fun getHistory(): List<String> = history.toList()
}
```

The `when` expression here is incredibly powerful. Because `Operation` is a `sealed class`, Kotlin's compiler knows all its direct subclasses. This means:

1.  **Exhaustive Checking:** If you forget to handle one of the `Operation` subclasses, the compiler will alert you, preventing a whole class of runtime errors.
2.  **Smart Casting:** Within each `when` branch, Kotlin automatically smart-casts `operation` to its specific subclass (e.g., `Operation.Add`), allowing direct access to its properties (`operation.a`, `operation.b`) without explicit casting.
3.  **Inline Error Handling:** We can embed specific error checks (like `operation.b == 0.0` for division) directly within the `when` branch, returning `CalculatorResult.Error` when an invalid condition is met.

We also maintain a `history` of operations using Kotlin's convenient [string templates](https://kotlinlang.org/docs/strings.html#string-templates), which is crucial for auditing financial transactions.

### Presenting Results with Extension Functions

Finally, for user-facing financial reports, we often need to format numbers to a specific decimal precision. An [extension function](https://kotlinlang.org/docs/extensions.html) on `Double` is perfect for this:

```kotlin
fun Double.format(decimals: Int = 2): String = "%.${decimals}f".format(this)
```

This allows us to call `.format()` directly on any `Double` value, making our display logic clean and reusable.

### Putting it all Together

Here's how you'd use our robust financial modeling engine:

```kotlin
fun main() {
    val calculator = Calculator()

    // Perform an addition
    val addResult = calculator.calculate(Operation.Add(100.50, 25.75))
    when (addResult) {
        is CalculatorResult.Success -> println("Portfolio value: ${addResult.value.format()}") // Output: 126.25
        is CalculatorResult.Error -> println("Error: ${addResult.message}")
    }

    // Attempt a division by zero
    val divResult = calculator.calculate(Operation.Divide(500.0, 0.0))
    when (divResult) {
        is CalculatorResult.Success -> println("Result: ${divResult.value.format()}")
        is CalculatorResult.Error -> println("Error: ${divResult.message}") // Output: Error: Cannot divide by zero
    }

    // Calculate a percentage
    val percentageResult = calculator.calculate(Operation.Percentage(1000.0, 12.5))
    when (percentageResult) {
        is CalculatorResult.Success -> println("12.5% of 1000 = ${percentageResult.value.format()}") // Output: 125.00
        is CalculatorResult.Error -> println("Error: ${percentageResult.message}")
    }

    // Review audit trail
    println("\n--- Audit History ---")
    calculator.getHistory().forEachIndexed { index, entry ->
        println("${index + 1}. $entry")
    }
}
```

This approach makes your code significantly more readable, maintainable, and less prone to runtime errors by leveraging Kotlin's type system to its fullest.

## Key Takeaways

1.  **Embrace `sealed class` for defining finite state machines or operation types:** This provides type safety and ensures all possible scenarios are considered at compile-time.
2.  **Utilize `when` expressions for exhaustive pattern matching:** When combined with `sealed class`, `when` forces you to handle every case, dramatically reducing unhandled state errors.
3.  **Implement `sealed class` for `Result` types:** Explicitly model `Success` and `Error` states to make error handling a first-class concern, leading to more robust and predictable APIs.

By mastering these Kotlin features, you can elevate your application's reliability and developer experience, especially when dealing with complex, critical operations.

---
Interested in diving deeper into Kotlin's powerful features for robust application development? Check out "Effective Kotlin: Best Practices to Write Better Code" on Amazon. [Link to the Amazon book here](https://www.amazon.com/Effective-Kotlin-Practices-Better-Code/dp/B0BP3F9C7D/).

---

**[Get the full Easy Mind guide on Amazon](https://www.amazon.com/dp/B0G8H4QX2L/)**

[← Back to Home](https://easy-mind-read.github.io/kotlin-read-the-first-pages/)