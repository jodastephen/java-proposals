# **Type conversion in Java**

Stephen Colebourne, 14th October 2025, v1

## **Summary**

This document teases apart the concepts of type checking and type conversion, proposing a new syntax that permits pattern matching to expand to primitives and value types.

## **Problem space**

The concept of "conversions" has an entire JLS chapter which includes 12 kinds of conversion and 6 conversion contexts. Recently, there has been significant work in [JEP 507](https://openjdk.org/jeps/507) and predecessors to enable full use of primitive types in pattern matching, proposed to be:

```
  long val = ...;
  switch (val) {
    case int i ->  // matches if value of long fits in an int
    case long v when v > 0 -> // guarded type pattern
    case long v -> // unconditional type pattern
  }
```

The current JEP is modelled around the concept that if you can cast from long to byte, you ought to be able to check if the cast is valid first. The JEP's approach is to extend the principle that a cast is usually preceded by an instanceof check to primitive types. Given the JEP has created some debate as to the approach taken, this document takes a step back to re-evaluate the problem space more widely.

For clarity, this document ignores edge cases that can obscure the underlying issues, such as inexact widening primitive conversions.

## **Casts and Conversion**

Consider the following two pieces of code:

```
  BigDecimal bd = new BigDecimal("123.45");
  Number num = bd;
  BigDecimal bd2 = (BigDecimal) num;
   
  int i = 56;
  long v = i;
  int j = (int) i;
```

At the surface level, there is a great deal of similarity between these two pieces of code. Assigning from BigDecimal to Number, and from int to long, is considered safe by the compiler. Whereas assigning in the opposite direction is unsafe, thus the compiler requires a cast. Dig deeper, and there are some significant differences.

At the JLS level, the distinction between the two is mostly expressed by the concepts of widening reference conversion, narrowing reference conversion, widening primitive conversion, narrowing primitive conversion, assignment contexts and cast contexts.

Taking a step back from the JLS, it seems reasonable to argue that reference and primitive casts are performing different jobs:

* For primitive types, the cast performs a *type conversion* from one primitive type to another, where the operation to perform is fully known at compile time and cannot fail at runtime  
* For reference types, the cast performs a runtime *type check* which may throw, where the operation does not alter the value (sometimes this is referred to in other languages as a type assertion)

Here is one [internet discussion](https://stackoverflow.com/a/1082478/38896) of the difference:  
> "Casting a reference doesn't change anything about the object it refers to. It only produces a reference of a different type pointing to the same object as the initial reference. Casting primitive values is different from casting references. In this case the values do change."

Reference casts:

* check if the object is of the requested type, or a subtype, or null  
* the cast makes no changes to the object  
* the check is performed at runtime  
* the check only examines the type of the object, not the value  
* if the check fails, an exception is thrown

Primitive casts:

* check if the value is permitted to be converted as per the JLS (a compile time check)  
* the cast creates a new value, with a different, if related, bit-pattern  
* no check is performed at runtime  
* if the cast compiles, it will succeed at runtime, even if data is lost

When examined carefully, the following can be observed:

* What happens:  
  * Reference casts check the object's runtime type. The cast completely ignores the object's value. In all cases, the object is completely unchanged by the cast.  
  * Primitive casts create an entirely different representation of the value. A number held as a long is very different to a number held as an int, even if the number has the same magnitude.  
* What happens on failure  
  * Reference casts can fail, causing an exception to be thrown at runtime.  
  * Primitive casts do not fail, even when data is lost. No exception is thrown.

It can clearly be seen that these two kinds of cast are very different.

So, how do developers keep these two different things clear in their head? Well, primitive types are well-known and have a lower-case type name, whereas reference types have an UpperCamelCase class name.

## **Type conversion**

So, how can we define *type conversion?*

For a typical developer, type conversion probably means manipulating business objects:

```
  CreatePersonRequest req = ...;
  InsertPersonDto dto = convertRequestToDto(req);
```

The type has been changed, with the result value derived from the input value.

Capturing this, suggests three criteria that can capture the concept of type conversion:

1) the type of the result is different to the type of the input  
2) the value is changed to match the new type  
3) the new value is, to some degree, equivalent to the old value

As an example, consider conversion from String to LocalDate. This matches the three type conversion criteria, with a different type and value, but where the value is related to the input.

Examining the current primitive type casts against the criteria:

1) "the type of the result is different to the type of the input" \- True  
2) "the value is changed to match the new type" \- True. When casting a long to an int, there is clearly a strong connection between the input and output, but they are absolutely distinct and different things in memory and representation.  
3) "the new value is, to some degree, equivalent to the old value" \- True. When casting a long to an int, the compiler does its best to keep the numeric value equivalent

Thus, primitive type casts meet the defined criteria for type conversion.

## **Value type casts**

A value type can implement an interface, unlike a primitive type. As such, type check casts will be needed to resolve the interface to the concrete type:

```
  Integral o = ...;  // new interface implemented by integral value types
  UnsignedInt i = (UnsignedInt) o; // type check
```

In this example, the cast is checking whether the runtime type of variable o is an UnsignedInt. This is the standard type check form of a cast.

However, it has also been suggested that type conversion casts could be used with value types:

```
  long v = 56L;
  UnsignedInt i = (UnsignedInt) v; // type conversion
```

It should be clear that there is a tension here. The same syntax is performing two different kinds of cast for the same type. Only one kind of cast applies in any given context. But there is confusion for the developer trying to read the code and unpick what is going on.

## **Summarizing the issue**

A cast in Java performs either a type check or a type conversion. Currently, developers can easily tell the difference between the two, because they can easily identify which types are primitive types, and thus which types are cast by type conversion. With Valhalla value types on the way, it is clear that this approach no longer works.

The only way developers would know what kind of cast is happening is to memorize which types are value types and which are reference types. This is very undesirable.

A separate distinction can be found between unsafe (lossy) type conversion and safe type conversion. For example, converting the value type Decimal to UnsignedInt:

```
  UnsignedInt i = (UnsignedInt) Decimal.of(56);   // safe
  UnsignedInt j = (UnsignedInt) Decimal.of(56.5); // unsafe (lossy)
```

Does it really make sense in Java, where the language places a premium on safety, to allow even more silently lossy type conversions?

To try and tackle this, and taking a step back, it can be noticed that a type check cast can throw an exception, while a type conversion cast does not. This observation leads to the proposed solution, a new kind of cast for Java.

## **Proposal**

This document proposes the introduction of a *type conversion cast* and a matching *type conversion pattern*.

### **Type conversion cast**

This document proposes a new kind of cast using the following syntax:

```
  (FromType~ToType)
```

The new cast operates as follows.

1) A type check is performed from the context type to the FromType, as though using a supertype/subtype relationship traditional cast. If it fails, ClassCastException is thrown.  
2) A check is performed to see if the value can be converted from FromType to ToType without loss. If it cannot be converted without loss, TypeConversionException is thrown.  
3) The value is converted, providing the result of the cast expression.

If FromType can be determined from the context (as it usually can), then it may be omitted:

```
  long v = 56L;
  int j = (~int) i;      // standard/recommended syntax
  int k = (long~int) i;  // unnecessary use of the full syntax
```

The full syntax is necessary to allow conversions where both a type check and a type conversion are necessary:

```
  Integral o = ...;  // new interface implemented by integral value types
  UnsignedInt i = (Long~UnsignedInt) o; // type check, then conversion
```

This code performs a type check from Integral to Long, followed by a type conversion to an UnsignedInt. It might throw ClassCastException or TypeConversionException. Given this, it can be seen that these two things are equivalent:

```
  UnsignedInt i = (Long~UnsignedInt) o;       // equivalent
  UnsignedInt i = (~UnsignedInt) ((Long) o);  // equivalent
```

It would be a compile time error for there to be no supertype/subtype relationship between the context type and FromType. Thus this would not compile:

```
  long v = 56L;
  int i = (int~short) o;  // INVALID!! int is not a subtype of long
```

No changes are proposed to traditional casts:

```
  long max = Long.MAX_VALUE;
  int i = (int) max;  // succeeds, but value is truncated
  int j = (~int) max; // throws TypeConversionException
```

For value types, narrowing primitive conversion would not be permitted with a traditional cast. Value types would only be permitted to use narrowing reference conversion \- the supertype/subtype type check. This ensures that UpperCamelCase casts are always about the supertype/subtype relationship:

```
  Integral o = ...;  // new interface implemented by integral value types
  UnsignedInt i = (UnsignedInt) o; // type check cast, allowed
  int j = (int) i;   // INVALID!! int is not a subtype of UnsignedInt
  int k = (~int) i;  // succeeds if conversion is exact, throws if not

  // INVALID!! UnsignedShort is not a subtype of UnsignedInt
  UnsignedShort m = (UnsignedShort) i;
  // succeeds if conversion is exact, throws if not
  UnsignedShort n = (~UnsignedShort) i;
```

This approach limits language support to value type conversions that are safe. Should developers need to perform a lossy conversion, such as converting Decimal 43.5 to UnsignedInt 43, they would need to use a method. In other words, only primitive types get language support for lossy conversion such as from long to int. This may appear like a serious constraint at first, but it is very much appropriate for Java, which places a premium on language features that protect developers from harm.

Note that nothing in this document precludes safe widening type conversions from and between value types without casts.

### **Type conversion pattern**

The dual of the type conversion cast is the *type conversion pattern*.

A type conversion pattern has the following syntax:

```
  FromType~ToType bindVariable
```

Matching a type conversion pattern proceeds in three stages:

1) A type check is performed from the context type to the FromType, as though using a supertype/subtype type pattern. If this fails to match, the whole pattern fails to match.  
2) A check is performed to see if the value can be converted from FromType to ToType without loss. if it cannot be converted without loss, the whole pattern fails to match.  
3) The value is converted and the result is bound to the specified local variable.

If FromType is the same as the pattern's target type, it may be omitted (because stage 1 of the matching process would always be true).

A type pattern continues to be defined as the pattern that checks supertype/subtype relationships. Primitive type patterns would be permitted at the root of switch, however they would only be allowed if the selector expression type is the same as the primitive type (because primitive types have no supertypes or subtypes):

```
  long val = ...;
  switch (val) {
    case int i ->             // INVALID! int is not a subtype of long
    case long v when v > 0 -> // guarded type pattern
    case long v ->            // unconditional type pattern
  }
```

In order to check whether the type conversion from long to byte or int is permitted, the new type conversion pattern must be used. In this example, the FromType of the pattern is inferred from the switch switch selector expression type, which would be the recommended best practice:

```
  long val = ...;
  switch (val) {
    case ~int i ->    // succeeds if long->int conversion is exact
    case long v ->    // unconditional type pattern
  }
```

The same approach applies when nested in records:

```
  record Id(long val) {}
  Id id = ...;
  switch (id) {
    case Id(~int i) ->  // succeeds if long->int conversion is exact
    case Id(long v) ->  // unconditional type pattern
  }
```

Type conversion patterns can also be used with instanceof:

```
  long val = ...;
  if (val instanceof ~int i) { ... }
```

As expected, this succeeds if the long-\>int conversion is exact.

As with casts, the full form is used if there needs to be both a type check and a type conversion:

```
  Integral o = ...;  // new interface implemented by integral value types
  switch (o) {
    case Long~int i -> // succeeds if o is Long and conversion is exact
    case Long v ->     // type pattern
    case Integral n->  // unconditional type pattern
  }
```

Were it ever to be considered desirable to build type conversions more fully into the language, the proposal handles it. For example, consider a future where it is possible to convert a LocalDate to and from a String using language-level type conversion:

```
  Object obj = ...;
  switch (obj) {
    case String~LocalDate date -> // type check and type conversion
    case Object o ->              // unconditional type pattern
  }
```

## **Summary**

This document examined the distinction between type checks and type conversions. Currently, reference type casts perform type checks, and primitive type casts perform lossy type conversions. With value types, this neat distinction no longer works, as value types can have interfaces, but also want to be converted like a numeric type.

In response, *type conversion casts* and *type conversion patterns* are proposed. Using a simple syntax, they allow type conversions to be clearly expressed in the language. The new kind of cast introduces TypeConversionException as the parallel to ClassCastException:

```
  long val = ...;
  int i = (int) val;  // long->int conversion, potentially lossy
  int j = (~int) val; // long->int conversion, throws if lossy
  switch (val) {
    case ~int k ->    // long->int conversion, succeeds if not lossy
    case long v ->    // unconditional type pattern
  }
```

The proposal leads to more readable code that is much easier to reason about. It is clear where there is a type check and where there is a potentially lossy type conversion. In particular, the ability to upgrade existing primitive casts to ones that throw if the conversion is lossy is hugely beneficial to writing safe code in Java.
