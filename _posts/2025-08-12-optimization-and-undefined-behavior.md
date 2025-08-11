---
title: "Optimization and UB"
date: 2025-08-12
categories: [software, C]
tags: [compilers, optimization, undefined-behavior]
---


As C programmers, we write human-readable source code, and a compiler translates it into machine-readable executables. But this "translation" is far from literal. A modern compiler is more like a master wordsmith than a simple translator; it polishes, rephrases, and restructures our logic to create a final program that is faster, smaller, and more efficient. This process is called **optimization**.

However, this process operates on a secret pact, a sacred agreement between you and the compiler. When this pact is broken, strange things begin to happen. This is the world of **Undefined Behavior (UB)**.

This post will take you on a journey from the shallow waters of basic optimization to the deep, mind-bending abyss of Undefined Behavior, explaining how these two concepts are fundamentally intertwined.


## Part 1: The Puzzle

Consider the following C program. Our goal is to assign `0x80000000` to a 32-bit signed integer, negate it, and then print both its value and whether it's greater than zero.

```c
#include <stdio.h>
#include <limits.h> // For INT_MIN

int main() {
    // On a 32-bit system, 0x80000000 is the most negative integer.
    int x = 0x80000000;

    printf("Initial x: %d\n", x); // Prints INT_MIN

    x = -x; // Negate x

    printf("Final x:   %d\n", x);
    printf("Is x > 0?  %d\n", x > 0);

    return 0;
}
```

Let's compile and run this code under two different conditions: once with no optimization (`gcc -O0 puzzle.c`) and once with the first level of optimization (`gcc -O1 puzzle.c`).

Here are the astonishingly different results:

**Compiled with `-O0` (No Optimization):**

```text
Initial x: -2147483648
Final x:   -2147483648
Is x > 0?  0
```

**Compiled with `-O1` (Basic Optimization):**

```text
Initial x: -2147483648
Final x:   -2147483648
Is x > 0?  1
```

Look closely. The final value of `x` is identical in both runs. Yet, the logical test `x > 0` evaluates to `false` (0) in the unoptimized version and `true` (1) in the optimized one\!

How is this possible? Is the optimizer broken? To solve this mystery, we need to dig deeper.

## Part 2: The Optimization Ladder

Compilers typically offer different levels of optimization, controlled by flags like `-O0`, `-O1`, `-O2`, etc. Let's climb this ladder step-by-step.

### `-O0`: No Optimization (The Ground Truth)

This is the baseline. The compiler does a direct, literal translation of your code.

  * **Philosophy:** "Do exactly what the programmer wrote, no more, no less."
  * **Result:** Every variable has a stable memory address on the stack. The execution flow precisely matches your source code, line by line.
  * **Impact:** The code is slow and large, but it's a paradise for debugging. This is your go-to for development.

### `-O1`: The First Rung (Low-Hanging Fruit)

The compiler starts to apply conservative, "low-risk, high-reward" optimizations that are fast and provide a good performance boost.

  * **Philosophy:** "Let's clean up the obvious stuff."
  * **Key Techniques:**
      * **Constant Folding & Propagation:** Pre-calculates constant expressions at compile time.
        ```c
        // Your code:
        int x = 10;
        int y = x * 2 * 3;
        // What the compiler sees:
        int y = 60;
        ```
      * **Dead Code Elimination:** Removes code that can never be reached.
        ```c
        // Your code:
        if (0) {
          // This is never executed
          printf("Hello!");
        }
        // What the compiler produces:
        // (Nothing)
        ```
  * **Impact:** A significant improvement in speed and size over `-O0` with a slight increase in compile time. Debugging becomes a bit harder as some variables might now live only in CPU registers.

### `-O2`: The Gold Standard (Aggressive but "Safe")

This is the standard for most release builds. The compiler performs nearly all major optimizations that don't involve a significant space-for-time tradeoff. It analyzes code on a much more global scale.

  * **Philosophy:** "Let's fundamentally restructure the code for maximum efficiency."
  * **Key Techniques:**
      * **Function Inlining:** Replaces a function call with the body of the function itself, eliminating call overhead.
      * **Instruction Scheduling:** Re-orders machine instructions to prevent CPU pipeline stalls.
      * **Loop Optimizations:** Moves loop-invariant calculations out of loops, and replaces expensive operations (like multiplication) with cheaper ones (like addition).
        ```c
        // Your code:
        for (int i = 0; i < 1000; i++) {
          array[i] = x * y; // x*y is invariant
        }
        // What the compiler might do:
        int temp = x * y;
        for (int i = 0; i < 1000; i++) {
          array[i] = temp;
        }
        ```
  * **Impact:** Usually the fastest code for general-purpose applications. Debugging becomes very difficult.

### `-O3`: The Daredevil (The Most Aggressive)

This level enables expensive optimizations that *might* provide more speed but can also increase code size significantly and sometimes even make the code slower.

  * **Philosophy:** "Let's try everything, even if it's risky."
  * **Key Techniques:**
      * **Aggressive Function Inlining:** Inlines even larger functions.
      * **Loop Unrolling:** Reduces loop overhead by duplicating the loop body.
      * **Auto-Vectorization (SIMD):** Uses special CPU instructions (like AVX or NEON) to process multiple data points in a single instruction, offering massive speedups for array-based computations.
  * **Impact:** Potentially the absolute fastest code, but with no guarantees. Code size can explode.

### `-Os`: The Minimalist (Optimize for Size)

  * **Philosophy:** "Every byte counts."
  * **Strategy:** Enables all `-O2` optimizations that do not increase code size, then applies further size-reduction techniques.
  * **Impact:** Produces the smallest possible executable. Essential for memory-constrained embedded systems.

### Summary Table

| Level   | Main Philosophy                         | Speed        | Size         | Compile Time | Debug Friendliness |
| :------ | :-------------------------------------- | :----------- | :----------- | :----------- | :----------------- |
| **-O0** | No optimization                         | Slowest      | Largest      | Fastest      | **Excellent**      |
| **-O1** | Basic, local optimizations              | Faster       | Smaller      | Fast         | Good               |
| **-O2** | Aggressive, "safe" global optimizations | Much Faster  | Can Increase | Slower       | **Poor**           |
| **-O3** | Speculative & expensive optimizations   | **Fastest?** | Large        | Slowest      | Very Poor          |
| **-Os** | Optimize for size                       | Fast         | **Smallest** | Slower       | Poor               |

## Part 3: The Compiler's Pact and Undefined Behavior

Now for the deep part. How can an optimizer make a program behave *differently*, as we saw with our `int x = 0x80000000;` example?

The answer lies in the C Standard, which is a contract between you and the compiler. This contract contains clauses for **Undefined Behavior (UB)**. UB refers to operations for which the standard imposes **no requirements**. If you write code with UB, you break the contract.

When the contract is broken, the compiler is free to do *anything*. Most importantly, it operates under this one golden rule:

> **The compiler is allowed to assume that Undefined Behavior will never occur.**

This assumption is a license to perform incredible optimizations.

### Case Study: Signed Integer Overflow

Let's revisit the classic example:

```c
#include <stdio.h>

void check(int x) {
    x = -x;
    printf("%d\n", x > 0);
}

int main() {
    int val = 0x80000000; // In 32-bit, this is INT_MIN
    check(val);
    return 0;
}
```

In a 32-bit signed integer system, `0x80000000` is `INT_MIN` (-2,147,483,648). Negating it causes a **signed integer overflow**, which is UB.

#### The `-O0` Result: The Hardware's View

Without optimization, the compiler generates a literal translation. The CPU's `NEG` instruction on `0x80000000` uses two's complement arithmetic, which results in `0x80000000`. This value, interpreted as a signed integer, is negative. Therefore, `x > 0` is false.

  * **Output:** `0`

#### The `-O1` Result: The Optimizer's View

With optimization, the compiler uses its "no UB" assumption:

1.  It sees the operation `x = -x` on a signed integer.
2.  It assumes overflow will *not* happen.
3.  Under this assumption, for any *valid* negative number `x`, `-x` is *always* positive.
4.  It sees that `x` starts as a negative number (`INT_MIN`).
5.  Therefore, it concludes that after `x = -x`, the expression `x > 0` **must logically be true**.
6.  It replaces the entire calculation and comparison with the constant `1`.

<!-- end list -->

  * **Output:** `1`

The optimizer didn't "get it wrong." It followed its rules perfectly, exposing the UB latent in our code.

#### A Deeper Question: Why is `printf("%d\n", x)` the same?

When we changed the code to print `x` directly, the output was the same for `-O0` and `-O1`. Why?

  * In `printf("%d\n", x > 0)`, the compiler could optimize the **result of the expression**. It found a logical shortcut to the value `1`.
  * In `printf("%d\n", x)`, the compiler must provide the **final value of `x`**. Since the standard doesn't define the result of the overflow, the compiler has no logical shortcut to a specific constant. Its most efficient path is to emit the hardware `NEG` instruction, which happens to produce the same result as the `-O0` compilation.

## Part 4: Do Well-Defined Programs Always Behave the Same?

This leads to our final question: If my code has **no UB**, will its behavior be identical across all optimization levels?

The answer is: **Yes, its final, observable behavior should be the same. But "behavior" is a slippery word.**

Here’s what can still change, even in well-defined code:

1.  **Performance & Resource Usage:** This is the entire point. Speed, memory usage, and power consumption will change. A program might work under `-O1` but crash with a stack overflow under `-O2` due to different stack usage patterns.
2.  **Floating-Point Precision:** Floating-point math is not perfectly associative (e.g., `(a+b)+c != a+(b+c)`). Optimizers may reorder operations, leading to minute differences in results.
3.  **Timing-Sensitive Code:** An empty loop intended for a software delay might be completely eliminated by the optimizer, as it has no observable effect. You must use `volatile` to prevent this.
4.  **Unspecified Behavior:** The standard sometimes provides a few valid options for behavior. For example, the evaluation order of function arguments (`printf("%d %d", f(), g())`) is unspecified. An optimizer might change the order, which matters if `f()` and `g()` have side effects.

## Conclusion

1.  **Optimization is a trade-off** between speed, size, compile time, and debuggability.
2.  **The C Standard is a contract.** Undefined Behavior breaks this contract, giving the compiler a license to make powerful—and sometimes surprising—optimizations.
3.  **Think like the compiler:** Assume UB will never happen and see if your code's logic can be "short-circuited." This will help you find latent bugs.
4.  For well-defined code, the logical output is guaranteed, but non-functional properties like timing and precision are not.

Write clean, standard-compliant code. Treat compiler warnings about questionable constructs as serious errors. Understand that the code you write is not the code that runs; it is merely the specification for the code that the optimizer will create. Happy coding!
