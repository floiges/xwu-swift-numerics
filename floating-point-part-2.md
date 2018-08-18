Concrete binary floating-point types, part 2
============================================

## Floating-point precision

Swift's floating-point types are intended to model the real numbers, but (as
with all such models) it is inexact. In fact, [almost all][ref 12-1] real
numbers cannot be represented exactly in binary floating-point format.

> _Background:_
>
> Even for beginners, some caveats can be important to keep in mind when working
> with binary floating-point values:
>
> * Many modest decimal fractions, such as 0.1, cannot be represented exactly.
> * Many integral values less than `greatestFiniteMagnitude` cannot be
>   represented exactly.
> * Basic arithmetic operations are often inexact; for example, `a + b - b == a`
>   does not always evaluate to `true`.

Though out of the scope of this article, some alternative choices for modeling
real numbers are as follows:

> __Binary floating point__  
> Pro: represents modest integers exactly, extremely fast hardware
> implementations, fixed memory size, and rounding errors are extremely
> uniform--they don't vary much with the number being represented.  
> Con: almost no decimal fractions have exact representations.
>
> __Decimal floating point__  
> Pro: represents modest integers and decimal fractions exactly, slower than
> binary but still faster than almost anything else, fixed memory size.  
> Con: at least an order of magnitude slower than binary floating point, and
> rounding error is significantly less scale-invariant.
>
> __Fixed-size rationals__  
> Pro: represents all modestly-sized integers and fractions exactly, fixed
> memory size, four basic operations are exact until you hit the limits of
> representation.  
> Con: denominators quickly grow too quickly to be used for non-trivial
> computations (this is usually a deal-breaker).
>
> __Arbitrary-precision rationals__  
> Pro: closed under four basic operations, represents most numbers most people
> will use exactly.  
> Con: representations get extremely large extremely quickly, large memory
> footprint if you have more than a few numbers.
>
> __Computable real numbers__  
> Pro: any number you can describe, you can work with.  
> Con: your numbers are now computer programs, and you arithmetic system is
> Turing-complete. Testing for equality is equivalent to solving the halting
> problem.
>
> [— Stephen Canon, Sep. 12, 2016][ref 12-2]

The Swift standard library provides binary floating-point types. Foundation
offers a decimal floating-point type discussed later; alternative
implementations that adhere to IEEE 754, such as those from [IBM][ref 12-3] (ICU
License) or [Intel][ref 12-4] (BSD License), can be wrapped in Swift.
Third-party libraries can provide a [fixed-width rational type][ref 12-5], and
existing libraries with C interfaces can be wrapped to provide an
arbitrary-precision rational type (or one can be implemented natively in Swift).

[ref 12-1]: https://en.wikipedia.org/wiki/Almost_all
[ref 12-2]: https://forums.swift.org/t/provide-native-decimal-data-type/4003/4
[ref 12-3]: http://speleotrove.com/decimal/decnumber.html
[ref 12-4]: http://www.netlib.org/misc/intel/
[ref 12-5]: https://github.com/xwu/NumericAnnex/blob/master/Sources/Rational.swift

### Striding

> _Background:_
>
> As would occur in any other language, repeated addition of a `stride` amount
> to a binary floating-point value can cause accumulation of rounding error:
> 
> ``` swift
> let stride = 0.1
> var x = 1.0
> 
> x += stride
> x += stride
> print(x - 1.2)  // 2.2204460492503131e-16
> 
> x += stride
> x += stride
> print(x - 1.4)  // 4.4408920985006262e-16
> ```

In Swift 4.0, `stride(from: 1.0, to: 2.0, by: 0.1)` avoids accumulation of
rounding error by instead computing the sequence of values as follows: `1.0`,
`1.0 + 1.0 * 0.1`, `1.0 + 2.0 * 0.1`, `1.0 + 3.0 * 0.1`...

Although this method of computing the sequence avoids an undesired artifact,
users should nonetheless be aware that the result will not be equivalent to that
obtained by repeated addition.

### Fused multiply-add

In Swift 4.0, the sequence `stride(from: -0.2, through: 1.0, by: 0.2)` does not
include the value `1.0`.

Despite avoiding accumulated rounding error from repeated addition, the
expression `-0.2 + 6.0 * 0.2` evaluates to `1.0000000000000002` due to
__intermediate rounding error__.

The IEEE 754 operation __fusedMultiplyAdd__ allows the same computation to be
performed in one step with a single rounding, improving the accuracy of the
result. This operation is available in Swift as the instance method
`addingProduct(_:_:)`.

For example:

```swift
-0.2 + 6.0 * 0.2               // 1.0000000000000002
(-0.2).addingProduct(6.0, 0.2) // 1
```

By altering the implementation of `stride(from:through:by:)` in Swift 4.1 to
use the fused multiply-add operation, the sequence
`stride(from: -0.2, through: 1.0, by: 0.2)` now includes the value `1.0`.

> The fusedMultiplyAdd operation was added to Swift as part of the Swift
> Evolution proposal [SE-0067: Enhanced floating-point protocols][ref 11-10].
> Intermediate rounding error in floating-point strides was eliminated in the
> Swift standard library in [late 2017][ref 12-6].

__A caveat about fused multiply-add operations:__ Although eliminating
intermediate rounding can improve the accuracy of results, it's not always the
case that an algorithm will benefit from its use. [As William Kahan points
out][ref 12-7], for a sufficiently large value `x`, we can see a surprising
result:

```swift
let x = 9007199254740991.0
(x * x - x * x).squareRoot()              // 0, as expected
(x * x).addingProduct(-x, x).squareRoot() // nan
```

The second result isn't zero because `x * x` cannot be represented exactly as a
value of type `Double`. In fact, `(x * x).addingProduct(-x, x)` actually
computes the amount by which `x * x` is inexact. Since `x * x` is rounded down,
the amount of inexactness is negative and its square root is not a number.

[ref 11-10]: https://github.com/apple/swift-evolution/blob/master/proposals/0067-floating-point-protocols.md
[ref 12-6]: https://github.com/apple/swift/pull/13007
[ref 12-7]: https://people.eecs.berkeley.edu/~wkahan/ieee754status/ieee754.ps

### Unit in the last place

> _Background:_
>
> The binary floating-point representation of a real number takes the form
> _significand_&nbsp;×&nbsp;2<sup>_exponent_</sup>. The __ulp__, or unit in the
> last place, of a finite floating-point value is the value of 1 in the least
> significant (i.e., last) place of the significand.

In general, the property `ulp` is equivalent to the distance between a finite
floating-point value and the nearest representable value greater in magnitude.
However, `greatestFiniteMagnitude.ulp` is finite even though the nearest
representable value greater in magnitude than `greatestFiniteMagnitude` is
`infinity`.

In Swift, the `ulp` of a non-finite value (whether infinite or NaN) is NaN. In
Java, by contrast, `Math.ulp(Double.POSITIVE_INFINITY)` evaluates to positive
infinity, and the same result is obtained when using negative infinity as the
argument.

As previously mentioned, the Swift equivalent to the C constants known as
`FLT_EPSILON` and `DBL_EPSILON` is a static property named `ulpOfOne`. That
name was chosen in order to prevent confusion surrounding the definition and
proper usage of the property. As the name suggests, `T.ulpOfOne` is equivalent
to `(1 as T).ulp` for any floating-point type `T`.

### Approximating π

In Swift, each floating-point type has a static property `pi` that provides the
value for π at its best possible precision, accurately _rounded toward zero_.

The purpose of rounding toward zero is to avoid angles computed in radians
from being rounded to a different quadrant. As a consequence,
`Float.pi < Float(Double.pi)` evaluates to `true`.

> It so happens that rounding π to the nearest representable value of type
> `Double` is equivalent to rounding π toward zero. However, rounding π to the
> nearest representable value of type `Float` is _not_ equivalent to rounding
> π toward zero.

### Subnormal values on 32-bit ARM

> _Background:_
>
> In the gap between zero and 2<sup><em>emin</em></sup>, where _emin_ is the
> minimum supported exponent of a binary floating-point type, a set of linearly
> spaced [__subnormal__ (or denormal) values][ref 12-8] are representable using
> a different binary representation than that of __normal__ finite values.
>
> On 32-bit ARMv7, the vector floating-point (VFP) co-processor supports a
> __flush-to-zero (FZ) mode__ for floating-point operations that is not
> compliant with IEEE 754. When the FZ bit is set, which it is by default,
> operations that would otherwise return a subnormal value instead return zero.
> Meanwhile, the NEON SIMD co-processor on ARMv7 always uses flush-to-zero mode
> regardless of the FZ bit.

For iOS platforms, it is possible to clear the ARMv7 FZ bit in C using inline
assembler; however, doing so has a negative effect on performance. As Swift does
not support inline assembler, it is not possible to disable flush-to-zero mode
from Swift.

__On 32-bit ARM, Swift floating-point types skip subnormal values.__ For
example, `(0 as Double).nextUp` evaluates to the least normal magnitude and
`(0 as Double).ulp` evaluates to zero.

[ref 12-8]: https://en.wikipedia.org/wiki/Denormal_number

### String representation

Until Swift 4.2, floating-point values were converted to strings using an
algorithm implemented in C++ that was based on the C11 function
[`vsnprintf`][ref 12-9].

A finite value's `description` and `debugDescription` could be different from
each other because the precision of `description` was less than that of
`debugDescription`.

> Specifically, the precision of `description` was
> [`std::numeric_limits<T>::digits10`][ref 12-10] and the precision of
> `debugDescription` was [`std::numeric_limits<T>::max_digits10`][ref 12-11].

The previous algorithm didn't guarantee __round-trip accuracy__ for
`description`, which is to say that `Double(x.description) == x` was not true
for all values of `x`. In addition, the algorithm would routinely include
extraneous digits in `debugDescription`:

```swift
// Swift 4.1
debugPrint(1.1) // 1.1000000000000001
```

In the past decade, new algorithms have been described that produce optimal
string representations of floating-point values with good performance. In 2015,
[Rust switched its implementation][ref 12-12] to a combination of the Grisu3 and
Dragon4 algorithms.

__Beginning in Swift 4.2, floating-point values are converted to strings using
[a variation of the Grisu2 algorithm][ref 12-13]__ that incorporates changes
outlined in the [paper describing the Errol3 algorithm][ref 12-14]. Swift's new
algorithm, which is implemented in C, shows better performance in benchmarks
than either Errol4 (the successor to Errol3) or Grisu3 with fallback to Dragon4.
For values other than NaN, `description` and `debugDescription` now give the
same result:

```swift
// Swift 4.2
debugPrint(1.1) // 1.1
```

Descriptions of NaN are discussed later.

[ref 12-9]: https://en.cppreference.com/w/c/io/vsnprintf
[ref 12-10]: https://en.cppreference.com/w/cpp/types/numeric_limits/digits10
[ref 12-11]: https://en.cppreference.com/w/cpp/types/numeric_limits/max_digits10
[ref 12-12]: https://github.com/rust-lang/rust/pull/24612
[ref 12-13]: https://github.com/apple/swift/pull/15474
[ref 12-14]: https://cseweb.ucsd.edu/~lerner/papers/fp-printing-popl16.pdf

---

Previous:  
[Concrete binary floating-point types, part 1](floating-point-part-1-rev-1.md)

Next:  
[Concrete binary floating-point types, part 3](floating-point-part-3.md)

_27 February–8 March 2018_  
_Updated 18 August 2018_
