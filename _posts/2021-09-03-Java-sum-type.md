---
layout: post
title: The quest for the perfect sum type in Java
---

The goal of this post is to find the perfect sum type implementation in Java.

As a hobbyist Rust developer, I'm used to [sum types](https://en.wikipedia.org/wiki/Tagged_union). 
However, I'm very sad that this notion doesn't exist in our company [^1] 's language of choice, Java.

## Why sum types ?

By quoting Wikipedia

> A tagged union, [or] sum type [...]  is a data structure used to hold a value that could take on several different, 
> but fixed, types.

Instead of "data structure", I would have said "type", as a sum type defines a type. Take the Rust code below:

```rust
struct ProductType {
  first: i32
  second: String
}

enum SumType {
  First(i32),
  Second(String)
}
```

Variables of type `ProductType` can hold an int AND a string, while values of type `SumType` can hold an int OR
a string.

A sum type is manipulated by checking its variant, usually through pattern matching. This is equivalent of doing
an `instanceof` call, but better, as the sum type expresses the types that you are allowed to match.

```java
class MakeString {
  public static String makeString(Object object) {
    if (object instanceof String) {
      return (String) object;
    }
    if (object instanceof Integer) {
      return String.valueOf(object);
    }
    // Must handle other cases, object could be anything :(
    throw IllegalArgumentException("Only String and int are supported");
  }
}
```

```rust
enum StringOrInt {
  String(String),
  Int(i32),
}

fn make_string(input: StringOrInt) -> String {
  match input {
  StringOrInt::String(value) => value,
  StringOrInt::Int(value) => value.to_string(),
  // No need to handle other types or use a wildcard, we have a fixed list of types 
  }
}
```

Being able to hold a value of one type over a predefined set of types offers a lot of expressiveness in the code,
especially when dealing with values [^2].

For example, the JSON specification describe data as a tree, where each node can be null, a boolean, a number, a string,
an array or an object. An array contains a list of nodes, and an object is a set key-value pairs, with keys being 
strings and values being nodes. A node can be expressed easily with a sum type, and operations on nodes can be written
simply:

```rust
enum JSonNode {
  Null,
  Bool(bool),
  Number(f64),
  String(String),
  Array(Vec<JSonNode>),
  Object(HashMap<String, JSonNode>),
}

// Return the value of the node if it is a string
fn get_string(input: JSonNode) -> Option<String> {
  match input {
  JSonNode::String(value) => Some(value),
  _ => None,
  }
}
```

For a language without sum types, it can be quickly cumbersome. We might need to rely on inheritance and the visitor
pattern, or do dynamic type checking.

Another strong point of sum types is error handling. By creating sum types with two variants, we can express the 
"happy case" and "unhappy case" as two variants. For example, in Rust, a function returning `Option<T>`
explicitly states that the returned value can be null. To access the underlying `T`, the caller must check if
the option is of variant `Some(T)`. In Java, where you are unsure if a returned object is null or not, you are
forced to program more defensively, or to inspect the implementation of the function you are calling.

## Java progress on sum types

Java has progressed a lot on supporting sum types the best it can. With Java 17, pattern matching will be in
preview mode. Combined with records and sealed classes, it will provide some kind of sum type. 

Records allows reduces boilerplate when dealing with value classes [^2]. It is also a great addition for code
expressiveness as it allows splitting value classes from service classes.

Sealed classes is the most important component. It allows limiting the number of implementors of an interface. 
This brings the "fixed set of types" part of sum types. This already makes switches or `instanceof` based checks 
exhaustive.

```java
public sealed interface JSonNode
  permits NullJSonNode, BoolJSonNode, NumberJSonNode, ArrayJSonNode, ObjectJSonNode 
{}
public record NullJSonNode() implements JSonNode {}
public record BoolJSonNode(boolean v) implements JSonNode {}
public record NumberJSonNode(double v) implements JSonNode {}
public record ArrayJSonNode(List<JsonNode> v) implements JSonNode {}
public record ObjectJSonNode(Map<String, JsonNode> v) implements JSonNode {}

static <T> T visit(JSonNode node, NodeVisitor<T> visitor) {
  if (node instanceof NullJSonNode) {
    return visitor.visitNull();
  } else if (node instanceof BoolJSonNode n) {
    return visitor.visitBool(n);
  } else if (node instanceof NumberJSonNode n) {
    return visitor.visitNumber(n);
  } else if (node instanceof ArrayJSonNode n) {
    return visitor.visitArrayl(n);
  } else if (node instanceof ObjectJSonNode n) {
    return visitor.visitObject(n);
  }
  // This chain of ifs is exhaustive, no need for an else
}
```

Finally, pattern-matching [^3] will bring more syntactic sugar, allowing code to look similar to the Rust version.

## Some constraints

Now that I have discussed the need of sum types, I will put some constraints on how to design one from scratch.

Indeed, I will not be waiting for Java 17, unreleased at the time this article is written, as I have code to write
today :upside_down_face:. I'm also unwilling to use preview features, to have a bit of stability guarantees. Our exploration will
the be based on:

- Java 14
- No preview features

This constraint prevent us from using many interesting features:

- Records, stabilized in Java 16
- Pattern matching for instanceof, stabilized in Java 16
- Sealed classes, stabilized in Java 17
- Pattern matching for switch, preview in Java 17

And our goal is to make the most expressive and convenient sum type. Bonus points will be awarded on how easy it
is to read the code, to maintain and evolve it.

The candidates below will have to implement a 3-variant sum type, containing a string, an int, and a list of variants.
This sum type have be used on two use-cases:

- Extracting a string from the string variant
- Displaying a representation as a string. This applies to any variant 

For the record, below is a Rust implementation

```rust
enum Type {
  String(String),
  Int(i32),
  List(Vec<Type>),
}

fn get_string(ty: Type) -> Option<String> {
  match ty {
  String(value) => Some(value),
  _ => None,
  }
}

fn display(ty: Type) -> String {
  match ty {
  String(value) => value,
  Int(value) => value.to_string(),
  List(value) => {
    let values = value.into_iter().map(display).collect::<Vec<_>>();
    values.join(", ")
  }
  }
}
```

It's done in 23 lines.

## Candidate 1: the visitor

Visitors are the most natural way of thinking about a type that can take one value in a limited, heterogeneous 
collection of types. In fact, I consider the visitor pattern to be the object-oriented variant of the sum type.

Let's write our sum type using a visitor, and try to use it.

```java
interface Type {
  <T> T accept(TypeVisitor<T> visitor);
}

interface TypeVisitor<T> {
  T visitString(String value);
  T visitInt(int value);
  T visitList(List<Type> value);
}

class StringType implements Type {
  private final String value;
  
  public StringType(String value) {
    this.value = value;
  }
  
  @Override
  <T> T accept(TypeVisitor<T> visitor) {
    return visitor.visitString(value);
  }
}

class IntType implements Type {
  private final int value;

  public IntType(int value) {
    this.value = value;
  }

  @Override
  <T> T accept(TypeVisitor<T> visitor) {
    return visitor.visitInt(value);
  }
}

class ListType implements Type {
  private final List<Type> value;

  public ListType(List<Type> value) {
    this.value = value;
  }

  @Override
  <T> T accept(TypeVisitor<T> visitor) {
    return visitor.visitList(value);
  }
}
```

The implementation is 47 lines long, not counting `equals` & `hashCode`, nor the fact that this code should be in 4 
files. It is usually admitted that visitor patterns are **verbose**.

Note that we have implemented a visitor that can return something. It can be incrementally better than the version
returning nothing, especially on readability.

Now let's use our visitor:

```java
static class TypeUtils {
  @Nullable
  public static String getString(Type type) {
    return type.accept(new TypeVisitor<>() {
      @Override
      public String visitString(String value) {
        return value;
      }

      @Override
      public String visitInt(int value) {
        return null;
      }

      @Override
      public String visitList(List<Type> value) {
        return null;
      }
    });
  }

  public static String display(Type type) {
    return type.accept(new TypeVisitor<>() {
      @Override
      public String visitString(String value) {
        return value;
      }

      @Override
      public String visitInt(int value) {
        return String.valueOf(value);
      }

      @Override
      public String visitList(List<Type> value) {
        return value.stream()
            .map(TypeUtils::display)
            .collect(Collectors.joining(", "));
      }
    });
  }
}
```

We have another 36 lines.

I have chosen to use inline visitor implementations, but visitors could have been put in different files, as classes. 
The advantage of inline visitor is its readability. Here, the two visitors are implementation details, and it's better 
to have them close to the body of the function they belong to.

Note that the first function takes the same space as the second, and requires two dummy implementations for variant
that should not be handled. This can be a bit annoying when we are manipulating a type with many variants, and are only
interested in a single variant.

To fix this, we could think of providing specialized methods, that resolve the variant for us. This bring us
to the candidate 2.

## Candidate 2: the encapsulating interface (v1)

As the visitor pattern is verbose, and requires an additional interface that is separated from the type definition,
why not get rid of it ? The goal of this candidate is to put more methods in the interface of `Type`, in order
to avoid defining the visitor.

```java
interface Type {
  default Optional<String> string() { return Optional.empty(); }
  default Optional<Integer> integer() { return Optional.empty(); }
  default Optional<List<Type>> list() { return Optional.empty(); }

  static Type string(String value) {
    return new StringType(value);
  }

  static Type integer(int value) {
    return new IntType(value);
  }

  static Type list(List<Type> value) {
    return new ListType(value);
  }
}

class StringType implements Type {
  private final String value;

  public StringType(String value) {
    this.value = value;
  }

  @Override
  public Optional<String> string() {
    return Optional.of(value);
  }
}

class IntType implements Type {
  private final int value;

  public IntType(int value) {
    this.value = value;
  }

  @Override
  public Optional<Integer> integer() {
    return Optional.of(value);
  }
}

class ListType implements Type {
  private final List<Type> value;

  public ListType(List<Type> value) {
    this.value = value;
  }

  @Override
  public Optional<List<Type>> list() {
    return Optional.of(value);
  }
}
```

This implementation is (sadly) even longer than the previous one, totaling 55 lines. 

However, it has some advantages. By putting the interface and it's implementors in the same file, we link them together. 
This will help the maintainer understand that this inheritance graph is less related to inheritance than to the 
definition of a single type.

This implementation also improve the API, exposing a single type to external users. We are only missing the `sealed`
keyword to prevent users from implementing the interface.

We can now try to use this second candidate.

```java
class TypeUtils {
  @Nullable
  public static String getString(Type type) {
    return type.string().orElse(null);
  }

  public static String display(Type type) {
    if (type.string().isPresent()) {
      return type.string().get();
    }
    if (type.integer().isPresent()) {
      return String.valueOf(type.integer().get());
    }
    if (type.list().isPresent()) {
      return type.list().get().stream()
          .map(TypeUtils::display)
          .collect(Collectors.joining(", "));
    }
    throw new IllegalArgumentException("Impossible");
  }
}
```

The implementation looks much better than the previous candidate, having only 17 lines. The wins mainly comes from 
the single variant use-case, that is handled easily. To match all variants, we rely on an equivalent of instanceof,
but without sealed interfaces, we must handle the wildcard case :confused:.

We might be able to enhance this and remove the wildcard case, but we will have to add more methods in the `Type`
interface. Introducing "Candidate 2 (v2)"

## Candidate 2: the encapsulating interface (v2)

In this implementation, we will add a new method, that is basically a functional version of the visitor pattern.

```java
interface Type {
  ...
  <T> T match(Function<String, T> matchString, Function<Integer, T> matchInteger, Function<List<Type>, T> matchList);
  ...
}

class StringType implements Type {
  ...
  @Override
  public <T> T match(Function<String, T> matchString, Function<Integer, T> matchInteger, Function<List<Type>, T> matchList) {
    return matchString.apply(value);
  }
}

class IntType implements Type {
  ...
  @Override
  public <T> T match(Function<String, T> matchString, Function<Integer, T> matchInteger, Function<List<Type>, T> matchList) {
    return matchInteger.apply(value);
  }
}

class ListType implements Type {
  ...
  @Override
  public <T> T match(Function<String, T> matchString, Function<Integer, T> matchInteger, Function<List<Type>, T> matchList) {
    return matchList.apply(value);
  }
}
```

This implementation adds 13 more lines, but allow us to give a pattern-matching feeling when implementing `display`.   

```java
public static String display(Type type) {
  return type.match(
    s -> s, // Match string, could be replaced by Function.identity()
    i -> String.valueOf(i), // Match int, could be replaced by String::valueOf
    l -> l.stream().map(TypeUtils::display).collect(Collectors.joining(", "))
  );
}
```

The display function is down to 7 lines from 11. It's not a big gain, but a lot of boilerplate 
(`instanceof` or `isPresent` / `get` calls) is gone.

The second version of candidate 2 is obviously bigger in terms of code, and longer to write, but offers the best API
(as for now). Maintaining the `match` function can be tedious though, a type with a lot of variants will create a
function with a lot of parameters.

## Candidate 3: the single class

Candidate 3's goal is to try to reduce the size of `Type` implementation. Instead of inheritance, we can use a single
class containing all the variants. We can enforce this class to have only one variant being set using constraints 
enforced by methods [^4]. This unfortunately means no enforcement by design, so the implementation needs to be reviewed
and tested correctly. Any new addition to this class must also be reviewed correctly.

```java
class Type {
  @Nullable
  private String string;
  @Nullable
  private Integer integer;
  @Nullable
  private List<Type> list;
  
  // Private constructor
  // Types are constructed with static factory methods below
  private Type(@Nullable String string, @Nullable Integer integer, @Nullable List<Type> list) {
    this.string = string;
    this.integer = integer;
    this.list = list;
  }
  
  public Optional<String> string() { return Optional.ofNullable(string); }
  public  Optional<Integer> integer() { return Optional.ofNullable(integer); }
  public  Optional<List<Type>> list() { return Optional.ofNullable(list); }
  
  public <T> T match(Function<String, T> matchString, Function<Integer, T> matchInteger, Function<List<Type>, T> matchList) {
    if (string != null) {
      return matchString.apply(string);
    }
    if (integer != null) {
      return matchInteger.apply(integer);
    }
    if (list != null) {
      return matchList.apply(list);
    }
    throw new IllegalArgumentException("Invalid");
  }

  // These factory methods ensure that only one variant is set
  static Type string(String value) {
    return new Type(value, null, null);
  }

  static Type integer(int value) {
    return new Type(null, value, null);
  }

  static Type list(List<Type> value) {
    return new Type(null, null, value);
  }
}
```

This implementation takes 45 lines, while offering the same API as the previous candidate. It has also the advantage
of only requiring a single implementation of `equals` & `hashCode`. It also looks more like a value class [^5].

However, this object does not have the same layout in terms of memory as the previous candidates. Previous 
implementations of `Type` only hold metadata and the fields of the specific variants, this `Type` class will always
have 3 members, one per variant. This can cause an impact on memory consumption, especially when variants are added
to the type.

This implementation also contains a wildcard throw in it's `match` implementation, as it is impossible for the Java
compiler to understand that the 3 ifs covers all the state that can be reached in this class.

## Candidate 4: the matcher

This last candidate is a departure from the previous patterns. In the previous candidates, we wanted to offer a
safe API to the developer, especially about being able to enforce the different variants that are allowed in
the sum-type.

This candidate relaxes this constraint, and will simply operate on `Object`s. Our goal will simply to offer a
pattern-matching like API for a subset of types. This can be useful on existing classes, without needing a heavy
refactoring.

```java
public class Matcher<T> {
  private final Function<Object, T> defaultMatcher;
  private final List<MatcherItem<T>> items = new ArrayList<>();
  public Matcher(Function<Object, T> defaultMatcher) {
    this.defaultMatcher = defaultMatcher;
  }
  public static <T> Matcher<T> withDefault(Function<Object, T> defaultMatcher) {
    return new Matcher<>(defaultMatcher);
  }
  public <U> Matcher<T> with(Class<U> clazz, Function<U, T> matcher) {
    items.add(new MatcherItem<>(clazz, o -> matcher.apply((U)o)));
    return this;
  }
  public T match(Object object) {
    // Through metaprogramming, we find if the input object
    // is compatible with the input class type. If it is the case, we call
    // the passed match function. Otherwise, we call the default match function.
    return items.stream()
        .filter(item -> item.isCompatible(object))
        .findFirst()
        .map(item -> item.matcher.apply(object))
        .orElseGet(() -> defaultMatcher.apply(object));
  }
  private static class MatcherItem<T> {
    private final Class<?> clazz;
    private final Function<Object, T> matcher;
    private MatcherItem(Class<?> clazz, Function<Object, T> matcher) {
      this.clazz = clazz;
      this.matcher = matcher;
    }
    private boolean isCompatible(Object object) {
      return object.getClass().isAssignableFrom(clazz);
    }
  }
}
```

This matcher implementation might look confusing, but is very convenient in practice:

```java
// Generics is very annoying in Java, especially when you want to use meta-programming.
// Here we wrap the generic class into something more concrete. this is a workaround
class ObjectList {
  public List<Object> objects;
}

class TypeUtils {
  @Nullable
  public static String getString(Object type) {
    // (bad) type inference force to do this. As we are unable to pass the return type
    // we need to explicitly create a parameter of type Matcher and annotate the return type
    Matcher<String> matcher = Matcher.withDefault(o -> null); // Here we retur null, Java cannot infer what's the return type
    return matcher.with(String.class, s -> s)
        .match(type);
  }
  
  public static String display(Object type) {
    return Matcher.withDefault(o -> "") // If the function returns the type already, everything compiles :)
        .with(String.class, s -> s)
        .with(Integer.class, i -> String.valueOf(i))
        .with(ObjectList.class, l -> l.objects.stream().map(o -> display(o)).collect(Collectors.joining(", ")))
        .match(type);
  }
}
```

This candidate can be annoying or delightful to use, depending on your opinion :slightly_smiling_face:. At least it is 
vaguely similar to the class based pattern matching that will be introduced in the future.

## Verdict

Let's try to do a more-or-less objective comparison of all these candidates

|                         | Implementation | Usage | Comments |
|-------------------------|----------------|-------|----------|
| Visitor                 | 1/5            | 3/5   | Known pattern, canonical implementation. Very verbose, in implementation and usage. |  
| Encapsulating interface | 3/5            | 4/5   | Quite good API. Still similar to the visitor pattern. Still a bit verbose in implementation. |  
| Single class            | 4/5            | 4/5   | Quite good API. Looks like a value class. Still a bit verbose in implementation. Implementation needs to be reviewed. Be careful of memory cost. |
| Matcher                 | 2/5            | 2/5   | Can be integrated into any codeline. Implementation is complex. Uses metaprogramming. Not really a sum-type. |

My personal preference goes to the single class design, simply because it resembles a value class. The encapsulating 
interface and visitor can also be good designs, as they are quite close to the future way of defining sum types in Java.
The matcher design is simply here just in case, but should be considered only as a last resort.

-----

[^1]: And many companies. Java is not hype, but is well-known production grade language.

[^2]: I tend to classify classes in two categories: values and services. Value classes carries values (hence the name),
      while service classes operate on values to produce new values. Example of value classes are dates, x-y coordinates, an 
      HTML tag, etc. 

[^3]: Preview in Java 17

[^4]: This is the *real* purpose of encapsulation. Encapsulation is about enforcing constraints, not about hiding 
      members and provide getters.

[^5]: Intuitively, value classes have "natural" constructors, getters, equals and hashCodes. These natural operators
      are implemented by delegating to operators on each member. In the case of an inheritance hierarchy, members might not
      be the same, so it's impossible to implement these natural operators.