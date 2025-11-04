# Embedded Records in Java

Stephen Colebourne, 26th October 2025, v1

## Summary

This document is a proposal for the addition of *embedded records* to the Java language, a mechanism that aims to unite classes and records. This concept might also be referred to as anonymous records or onion records.

## Problem space

Java has had *records* in the language for a number of years now.
They shine when there is a need to express data in a fully transparent way.

The key record features are:

* shallowly immutable
* a short concise syntax
* a name (these are nominal tuples, not structural tuples)
* each record component has a name, and that name is part of the API
* record components are ordered (unlike fields in a regular class)
* a guaranteed canonical constructor and deconstructor
* compact constructor for validation

Taken together, these are a mighty powerful set of features, so long as the developer is willing to buy into the constraints:

* everything is open and transparent (no encapsulation)
* no ability to have lazy/cached/derived fields
* deep immutability requires work by the developer
* the getter syntax differs from long-standing practice with beans
* tools that work with beans may work differently with records

Another constraint is the lack of wither methods as an alternative to setters, and the lack of buliders.
This document will not address those issues, as solutions have already been outlined elsewhere.

Not everyone is using records, and that makes sense.
A record is not the same as a bean, even if the two do have significant overlaps.
Complicating this further is that it is difficult to fall back from a record to a class,
partly due to the syntax difference, partly due to the differnce in getters, and partly due to
differences in how frameworks handle beans and records.

Looking forward, work on [Serialization v2.0](https://www.youtube.com/watch?v=guF2NvgJIN8) is progressing well.
It is now planned to use records to represent data during the serialization process.
This is a highly sensible move - records are a great match for data passing during serialization.
Howver, it effectively requires a mechanism to expose classes as records.

Another possible language feature that hasn't progressed so far is [user-defined deconstruction](https://mail.openjdk.org/pipermail/amber-spec-experts/2023-March/003766.html).
Various syntaxes have been suggested over the years, but none has been entirely convincing.
Ultimately, deconstruction has a backwards data flow from everything else in Java, and any syntax will naturally be difficult to follow.

So, the problem space is that the industry has many millions of beans that fundamentally represent data.
While many of the beans could be represented by records, many millions cannot.

Ultimately, what Java needs is a way to "Code like a Class, express Data like a Record".

## Degrees of freedom

The problem space has various of degress of freedom to capture.
The language is not generally the place for a vast number of options, but some will be needed to capture the use cases.

**Mutable vs Immutable**. Classes can be mutable or immutable, and both types have a need to express their state as Data.

**Encapsulated vs Open**. Classes may want to fully encapsulate their state, only providing access to frameworks like serialization. But classes might also want to allow their state to be fully, or partially, accessed from other parts
of Java code, like most beans do.

**Naming conventions**. Classes may use any naming convention, with the two main options being record-style
and bean-style getters/setters/withers. Encoding bean-style into the language is not considered to be acceptable.

**Additional fields**. Classes are not limited to only hold fields representing their state.
They may also have fields representing derived/cached values, or for other purposes.
The state used exposed to the application might also be different to the state exposed to frameworks (serialization).

It turns out that most of these degrees of freedom do not require special language treatment
in any new feature because classes already cover most of the needs.

## Proposal

This proposal expresses both semantics and syntax for the purpose of explanation.
Readers should not assume that this is the only viable approach to the problem space, or the only viable syntax.
However, on balance, it is better to see the concepts expressed via a complete proposal.

### Embedded records

This document proposes that any concrete class may contain an *embedded record*:

```java
  public class Person {
    record(String name, LocalDate dob) {}
  }
```

As can be seen, the class has an embedded record, with no explicit class name.
In reality, it would have a well defined name such as `Person$Record`.

The record is expected to hold all the significant state of the class.
It exists to provide a standardised way to express the class as pure data,
but without losing the flexibility benefits that classes offer.

The embedded record itself is more or less a normal record.
It would be possible to write methods within the record.
For example, it may well be useful for the record to have a validating constructor,
or to override generated methods for arrays.
It is probably less useful to write methods in the record that do not replace generated methods,
put it would be permitted.

The class refers to the record as though it had a private generated field named `record` of type `Person$Record`.
With value types, it is expected that all embedded records would be value records,
thus the overhead of a second binary class would be minimal.
A simple private constructor would also be generated, allowing the class to be created from the record.

```java
  // developer writes
  public class Person {
    record(String name, LocalDate dob) {}
  }
  // compiler generates
  public class Person {
    private static value record Record(String name, LocalDate dob) {}
    private Record record;
    private Person(Record r) { this.record = r; }
    // see below for other generated methods
  }
```

### Generated methods

For a developer, the biggest convenience a record provides are the generated methods.
With this proposal, the compiler generates methods on the class based on the methods generated in the embedded record:

```java
  // developer writes
  public class Person {
    record(String name, LocalDate dob) {}
  }
  // compiler generates
  public class Person {
    private static value record Record(String name, LocalDate dob) {}
    private Record record;
    private Person(Record r) { this.record = r; }
    public Person(String name, LocalDate dob) { this.record = new Record(name, dob); }
    public String name() { return record.name(); }
    public void name(String name) {record = new Record(name, record.dob()); }
    public LocalDate dob() { return record.dob(); }
    public void dob(String dob) {record = new Record(record.name(), dob); }
    public boolean equals(Object obj) {
      return obj instanceof Person other && record.equals(other.record);
    }
    public int hashCode() { return record.hashCode();
    public String toString() { return record.toString(); }  // this would be tweaked to fix the class name
  }
```

As shown, the constructor, getters, equals, hashCode and toString would all be generated on the class.
In addition, setters are generated on the class.
These setters would follow the style of record getters, and not be prefixed by 'set'.
(Note that the record remains immutable, and the setters are not present on the record).

If a method is defined on the class with a signature that matches what the compiler would have generated,
the user-defined method in the class takes precedence. Note that it is possible for the user-defined
method to be private, effectively blocking the generated method from the public API.

### Variations

Two keywords may be used in front of the embedded record - private and final.

**Private**. If the embedded record is declared private, it is encapsulated within the class.
No methods would be generated on the class.
Only the record constructor would be generated.
The class is then free to expose as much or as little of the record as desired.

**Final**. if the embedded record is declared final, the generated field in the class would be final.
No setters would be generated on the class, but the other methods would still be generated.
This is necessary to support immutable beans.

It can be noted that the public API of a class with a final embedded record is identical to that of the equivalent record:

```java
  // these two have the same public API
  public class Person {
    final record(String name, LocalDate dob) {}
  }
  public record Person(String name, LocalDate dob) {}
```

### Construction and Deconstruction

Non-private embedded records cause an all-args constructor to be generated on the class based on the record components.

Classes with embedded records would support the same compact constructor syntax as records.
In the compact constructor, developers can assume that the embedded record has been assigned,
but all other fields are available to be set.
This makes it relatively easy to handle derived or cached fields:

```java
  public class Person {
    // developer writes
    record(String name, LocalDate dob) {}
    private final int cachedHashCode;
    // compact constructor, which integrates into the generated constructor (not shown)
    Person {
      cachedHashCode = record.hashCode();
    }
    // hashCode method is manually written, blocking the method that would normally be generated
    public int hashCode() { return cachedHashCode; }
    // compiler generates constructor, accessors, equals and toString, but not hashCode
  }
```

If the record is non-private, it is proposed that the compiler also generates a deconstructor on the class.
It should be noted that this does not require any syntax to be defined for deconstructors.
Potentially, this means that the language never has to tackle the problem of a general-purpose syntax for deconstructors.

### Beans

One common set of methods that developers might want to support is getters and setters.
In this proposal, the language provides no support for generating getters and setters.
Developers would need to write code similar to that written today.

```java
  public class Person {
    // developer writes
    private record(String name, LocalDate dob) {}
    public Person() { this.record = new Record(null, null); }
    public String getName() { return record.name(); }
    public void setName(String name) { record = new record(name, record.dob); }
    public LocalDate getDob() { return record.dob(); }
    public void setDob(LocalDate dob) { record = new record(record.name, dob); }
    public boolean equals(Object obj) {
      return obj instanceof Person other && record.equals(other.record);
    }
    public int hashCode() { return record.hashCode();
    public String toString() { return record.toString(); }
  }
```

It can be seen that the embedded record adds relatively little in this case.
But it does fulfil the requirement to clearly define how the bean exposes it's data,
something that greatly simplifies frameworks.

## Discussion and Alternatives

The proposal as written is a relatively simple extrapolation of existing languages features to meet the goal
of exposing the data in classes as records.
The author of a class simply decides what their core state is, rewrites that state as an embedded record,
and adapts the class as necessary.
The class itself loses very little flexibility in doing so,
although the intention is that the embedded records feature exists for classes that represent data, not just any class.

### Explicit method generation

One alternative approach would be to provide more control over method generation.
This might be referred to as 'exporting' the method from the embedded record to the class.
The primary reason to do so would be to make the bean use-case simpler.
This might look something like this:

```java
  public class Person {
    // developer writes
    record(
        String name exports getName:setName,
        LocalDate dob exports getDateOfBirth:setDateOfBirth)
            exports constructor, deconstructor, equals, hashCode, toString {}
    // compiler generates each item that was specified
  }
```

This alternative approach has explicit syntax via `exports` to specify which methods are exported, and under what names (for the accessors).

On the up side, the developer gets a lot more control, and there would be no need for the `private` modifier.
On the down side, there is a lot more complication where the main use case, beans, can be viewed as legacy.
On balance, the embedded records proposal, which generates methods via an all-or-nothing approach, is a much simpler approach, even if less flexible.

### Conversion records

Rather than focussing on altering the class to be based on a record, an alternate design centre could be to focus on conversion to/from a record.
This might look something like this:

```java
  public class Person {
    // developer writes
    private String name;
    private LocaLDate dob;
    public record Person(String name, LocalDate dob) {
      this.name = name;
      this.dob = dob;
    }
    public record toState() {
      return new record(name, dob);
    }
    // developer also writes getters, setters, equals, hashCode, toString
    // compiler generates an anonymous record
  }
```

This alternative approach uses the `record` keyword to identify the matching pair of methods for construction and deconstruction.
Behind the scenes, a record would be generated for the purpose of conversion.
The constructor provides the definition of the record (its shape) and a way to create the class from the record.
The deconstructor provides a way to extract the data from the class into the record.

On the up side, the developer gets a simple additive way to expose any class as data in a record.
On the down side, the developer gets none of the benefits of compiler-generated methods.
On balance, the embedded records proposal, which provides method generation, is a more complete and useful approach

### Extending annotation processing

Annotation processing (JSR-269) is a core part of mdern Java.
However, it has the restriction that code cannot be generated in the same class as the annotation being processed.
While sensible, this has resulted in sub-optimal outcomes, such as Lombok (which work around the limitation)
and Immutables (which generate more classes than the developer actually wants).

Embedded records offer an opportunity to revisit JSR-269.
What would be needed is a different kind of annotation processor, an "annotation export processor".
If the annotation is present, the processor would be invoked whenever the class-level generated methods are generated.
The processor would have the ability to return the set of methods to be generated into the class.
It should be self-evident that an annotation export processor could easily be written to generate methods according to
the bean conventions - getters and setters - or any other convention, such as Swing or JavaFX.

On the up side, annotation export processors would close a gap in annotation processing, removing the need for
some of the less-than-ideal current tooling. Genuine beans could be generated without the language needing to support it.
On the down side, there is a degree of additional complexity, which mostly exists to support the legacy bean conventions.
On balance, this would be worth exploring further.

## Summary

This document examines a recurring problem space for Java developers - classes (beans) that express data, but cannot be represented as records.

It proposes *embedded records* as a solution to the problem space.
The key aspect of the design is to provide method generation on classes as an extrapolation of that on the embedded record,
with a limited number of variations, rather than an overly complex full-featured generation approach.

It aims to push developers towards the positive aspects of records, while still allowing much of the flexibility that classes offer.
It dovetails well into the work of serialization 2.0 and user-defined deconstruction.
Fundamentally, it allows any class that has a data-based repesentation to expose that data in a standard way - a record.
