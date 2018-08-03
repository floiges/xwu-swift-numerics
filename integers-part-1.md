Concrete integer types, part 1
==============================

## Introduction

The Swift standard library provides ten concrete integer types, all defined as
value types wrapping [LLVM primitive types from the `Builtin` module][ref 1-1]:

```swift
@_fixed_layout
public struct UInt8 /* ... */ {
  /* ... */
  public var _value: Builtin.Int8
  /* ... */
}
```

One signed type, `Int`, and one unsigned type, `UInt`, each has bit width equal
to the platform's native word size. It is possible to determine the bit width of
`Int` or `UInt` via the static or instance property named `bitWidth`, but there
is no platform condition available in Swift to evaluate pointer bit width at
compile time.

The four remaining signed types are of explicit bit width (`Int8`, `Int16`,
`Int32`, `Int64`), as are the four remaining unsigned types (`UInt8`, `UInt16`,
`UInt32`, `UInt64`). [LLVM does support 128-bit integer types on all
platforms][ref 1-2], but this support is not surfaced by the Swift standard
library.

The built-in standard library integer types simultaneously model integers and
the sequences of bits that are used to represent them. In other words, some
functions (such as those that perform basic arithmetic) operate on the integer
value that is represented, while other functions operate on the binary
representation of those values.

All built-in standard library integer types use [two's complement][ref 1-3]
representation for signed values, although standard library protocols don't
preclude a hypothetical arbitrary-width integer type ("BigInt") from using
[sign-and-magnitude][ref 1-4] or another representation internally.

The basic arithmetic infix operators `+`, `-`, `*`, `/`, `%`, and the prefix
operator `-`, have behavior that will be largely familiar to users of other "C
family" languages. __In Swift, however, these operators trap on integer
overflow.__ It can sometimes be overlooked that `-T.min` overflows if `T` is a
signed (and fixed-width) integer type. As will be discussed below, other
functions are available that provide alternative overflow behavior.

Each of the basic arithmetic operators have a corresponding mutating (in-place)
counterpart. For the infix operators, those are the assignment operators `+=`,
`-=`, `*=`, `/=`, and `%=`. For prefix operator `-`, the mutating counterpart is
spelled as the instance method `negate()`. These functions also trap on integer
overflow.

[ref 1-1]: https://swift.org/compiler-stdlib/#standard-library-design
[ref 1-2]: https://github.com/rust-lang/rfcs/blob/master/text/1504-int128.md
[ref 1-3]: https://en.wikipedia.org/wiki/Two%27s_complement
[ref 1-4]: https://en.wikipedia.org/wiki/Signed_number_representations#Signed_magnitude_representation

## Integer literals

An integer value can be represented in Swift source code as an __integer
literal__. The same value can be written in any of four different bases; a
prefix is used to indicate bases other than 10:

```swift
let x = 42        // Decimal (base 10).
let y = 0b101010  // Binary (base 2).
let z = 0o52      // Octal (base 8).
let a = 0x2a      // Hexadecimal (base 16).
```

In Swift, leading zeros do not affect the base of an integer literal or its
represented value. Underscores (`_`) can be used to group digits in numeric
literals (e.g., `100_000_000`); they too have no effect on the value that is
represented. 

> In many other "C family" languages, leading `0` is used as an octal prefix.
> That is, numeric constants written with a leading `0` are [interpreted in base
> 8][ref 2-1]. This has been a source of unintended error and confusion, and
> Swift does not perpetuate the convention.

Negative values can be represented by prepending the hyphen-minus character
(`-`). This is considered to be part of the integer literal. In other words, the
expression `-42` is lexed as a single value, not as a call to the prefix
operator `-` with `42` as its operand.

> Should a distinction become necessary, the expression `-(42)` is lexed as a
> call to the prefix operator `-`. The distinction can be observed when negating
> a literal `0` while working with floating-point types that support distinct
> representations for `+0.0` and `-0.0`:
>
> ```swift
> let x: Double = -0
> x.sign // .plus
> 
> let y: Double = -(0)
> y.sign // .minus
> ```
>
> In the top example, a built-in 2048-bit integer is initialized to the value
> `-0`, but integer types do not support signed zero and the sign is ignored.
> Then, based on the type annotation, the integer value is converted to a
> positive floating-point value.
>
> In the bottom example, a built-in 2048-bit integer is initialized to the value
> `0`. Based on the type annotation, the value is converted to a positive
> floating-point value, then the floating-point prefix operator `-` is called to
> change the sign.

Note that the Swift standard library defines the prefix operator `+` for
symmetry but does __not__ consider a prepended `+` to be part of an integer
literal. Therefore, `+42` __is__ lexed as a call to the prefix operator `+` with
`42` as its operand.

In spite of these considerations, the member expression dot operator (`.`) binds
more tightly than both the prefix operator `-` and the prepended literal `-`.
For example:

```swift
-(42.trailingZeroBitCount) // -1
(-42).trailingZeroBitCount //  1
-42.trailingZeroBitCount   // -1
```

[ref 2-1]: https://blogs.msdn.microsoft.com/oldnewthing/20140116-00/?p=2063

### Type inference

In Swift, literals have no type of their own. Instead, the type checker attempts
to infer the type of a literal expression based on other available information
such as explicit type annotations:

```swift
let x: Int8 = 42
```

Besides using an explicit type annotation, the __type coercion__ operator `as`
[(which is to be distinguished from __dynamic cast__ operators `as?`, `as!`, and
`is`)][ref 3-1] can be used to provide information for type inference.

```swift
let x = 42 as Int8
```

In the absence of other available information, the inferred type of a literal
expression defaults to `IntegerLiteralType`, which is a type alias for `Int`
unless it is shadowed by the user:

```swift
typealias IntegerLiteralType = Int32
let x = 42
type(of: x) // Int32
```

> The following caveat applies to current versions of Swift. It __will not__
> be applicable after changes described in [SE-0213: Integer initialization via
> coercion][ref 3-2], which was [implemented in July 2018][ref 3-3], are
> included in a future Swift release.

__A frequent misunderstanding__ found even in the Swift project itself concerns
the use of a __type conversion__ initializer to indicate the desired type of a
literal expression. For example:

```swift
// Avoid writing such code.
let x = Int8(42)
```

This usage frequently gives the intended result, but the function call does
__not__ provide information for type inference. Instead, this statement creates
an instance of type `IntegerLiteralType` (which again, by default, is a type
alias for `Int`) with the value `42`, then __converts__ this value to `Int8`.

The distinction can be demonstrated as follows:

```swift
let x = 32768 as Int16
// Causes a compile time error:
// integer literal '32768' overflows when stored into 'Int16'

let i = 32768 as Int
let y = Int16(i)
// Causes a **runtime** error:
// Not enough bits to represent a signed value

let z = Int16(32768)
// Causes a **runtime** error:
// Not enough bits to represent a signed value
```

Differences in diagnostics are unlikely to be of interest to most users.
However, the same misunderstanding with floating-point types can produce
different results due to unintended rounding error:

```swift
let a = 3.14159265358979323846 as Float80
// 3.14159265358979323851

let b = Float80(3.14159265358979323846)
// 3.141592653589793116
```

[ref 3-1]: https://github.com/apple/swift-evolution/blob/master/proposals/0083-remove-bridging-from-dynamic-casts.md
[ref 3-2]: https://github.com/apple/swift-evolution/blob/master/proposals/0213-literal-init-via-coercion.md
[ref 3-3]:
https://github.com/apple/swift/pull/17860

## Conversions between integer types

Five different initializers are available to convert between standard library
integer types. A value `source` of type `T` can be converted to a value of type
`U` as follows:

1. __`U(source)`__  
   Converts the given value if the result can be represented exactly as a value
   of type `U`.  
   Otherwise, a runtime error occurs.

1. __`U(exactly: source)`__  
   _Failable initializer._  
   Converts the given value if the result can be represented exactly as a value
   of type `U`.  
   Otherwise, returns `nil`.

1. __`U(clamping: source)`__  
   Converts the given value to the closest representable value of type `U`.  
   If `source > U.max`, then the result is `U.max`.  
   If `source < U.min`, then the result is `U.min`.

1. __`U(truncatingIfNeeded: source)`__  
   Creates a new value of type `U` from the binary representation in memory of
   `source`.  
   When `T` and `U` are not of the same bit width, the binary representation of
   `source` is [truncated or sign-extended][ref 4-1] as necessary.

1. __`U(bitPattern: source)`__  
   _Available only for conversion between signed and unsigned types of
   explicitly the same bit width._  
   Creates a new value of type `U` with the same binary representation in memory
   as that of `source`.

[ref 4-1]: https://developer.apple.com/documentation/swift/int/2926530-init

### A note about init(truncatingIfNeeded:)

In previous versions of Swift, the same initializer was named
`init(extendingOrTruncating:)`. It was renamed to emphasize the potentially
lossy semantics of truncation over the lossless semantics of sign-extension.

Indeed, if `T.bitWidth < U.bitWidth`, then
`U(truncatingIfNeeded: source) == source` __except in one scenario__: If
`source < 0` and `U` is an __unsigned__ type, the binary representation of
`source` is padded with leading one bits and the result is equivalent to
`0 &- U(truncatingIfNeeded: -source)`.

---

Next:  
[Concrete integer types, part 2](integers-part-2.md)

_27 February–5 March 2018_  
_Updated 3 August 2018_
