# Member Access Tee Expressions

_AKA "Do expressions"._

## Summary

Provide the ability to chain methods that return `void` by offering a way to tee the member access chain.

```cs
// in the following code, Sort, Insert, and RemoveAll return void.
var myGrandchildList = MyObjectTreeThing.SomeChild.GrandchildList
  do {
    .Sort(),
    .Insert(0, aNewMember),
    .RemoveAll(el => el.SomePredicate())
  };
```

## Motivation

Realistically not everything is going to be immutable.  Mutation-heavy methods happen.

When working with long sets of void-returning mutations code can become very verbose.  They're also incompatible with many expression-only conveniences since they require multiple statements.  This creates the desire for self-returning "fluent" and "builder" interfaces.  Those are problematic because they create ambiguity about whether a given method is returning a new immuitable object with the change made, or if the object is being mutated in place.  Returning `void` is much more explict, but creates tedium which creates the impetus for the "everything returns `this`" pattern.

Instead, we provide an operator to get back `this` even after calling a void-returning method members.  This allows the object's API to be honest that its methods are wholly mutations, while the user maintains the convenience of expression-like behaviour.

Initializers and `with` expressions have demonstrated a similar usefulness for being able to use "setters" on newly-constructed objects within an expression, and the `do` can complement them.

```cs
public class MyClass
{
  public ChildClass BuildChild()
    => new ChildClass(this)
      {
        FooMember = "Foo",
        BarMember = "Bar"
      }
      do { .Initialize() };
}
```

## Detailed design

"Do expressions" would be implemented as effectively syntactic sugar over a new built-in generic extension method `public static T DoActionAndGoBack<T>(this T target, Action<T> action)` method:

```cs
namespace System.MemberAccessTeeOperator;
public static class MemberAccessTeeOperatorObjectExtensions
{
  public static T DoActionsAndGoBack<T>(this T target, params Action<T>[] doActions)
  {
    foreach(var doAction in doActions)
    {
        doAction(target);
    }
    return target;
  }
}
```

They are similar to `with` expressions except that they 
- do not imply a copy operation
- are not restricted to records, and
- allow full use of the members, instead of being constrainted to assignment expressions on properties.

The odd syntax of starting each member access with `.` makes sense when considering that these are not simple lists of assignment expressions as in `with` expressions or object initializers.

### Syntax

Referencing, [12.8.7.1](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/expressions.md#12871-general)

"Do Expressions" can be injected before any member access:

```
do_operator_expression
  : primary-expression 'do' '{' do_expression_list '}'
  ;

do_expression_list
  : do_expression (',' do_expression)*

do_expression
  : member_do_expression
  | element_do_expression

member_do_expression
  : '.' identifier

element_do_expression
  : '[' argument_list ']'
```

Where `member_do_expression` and `element_do_expression` can act as `primary_expression` in other expressions (such as assignment, invocation, or member-access).

The `do_operator_expression` resolves to the value of its `primary-expression`.

We're reusing the `do` keyword for this feature because it is only currently used in `do...while` loops, and so there should not be any ambiguity or conflict from using it here.  The leading expression and the lack of a trailing `while` statement should make it clear to the reader that this is not a `do...while` loop.

### Do Expressions using Assignment

Assignment to Properties, Fields, and Elements could also make use of `do`, particularly when doing many assignments into indexers.

example usage:

```cs
var myDictionary = myParentObject do
  {
    .SomeStringChild = "example string"
  }
  .SomeDictionaryChild do
  {
    ["Sum"] += someVar,
    ["One"] = 1,
    ["Two"] = 2,
    ["Three"] = 3,
    ["Four"] = 4
  }
```

## Other Possible Characters for the Operator

Ideas worth considering, which AFAIK would not be valid character/keyword combinations in current C#:
- `do {}` (preferred)
- `in {}` eg `var myGrandchildList = MyObjectTreeThing.SomeChild.GrandchildList in { .Sort() }`
- `.do {}` eg `var myGrandchildList = MyObjectTreeThing.SomeChild.GrandchildList.do{ .Sort() }`
- A combination?  `in do`?

`.do` may be a bit more visually familiar, since it's clearly an expression chain, but it's very inconsistent with `with`.  But on the plus side, `.do` would avoid the possibility that a misplaced semicolon would convert a `do` loop into a `do` expression or vice versa.

```cs
var myDictionary = myParentObject
  .do
  {
    .SomeStringChild = "example string"
  }
  .SomeDictionaryChild
  .do
  {
    ["Sum"] += someVar,
    ["One"] = 1,
    ["Two"] = 2,
    ["Three"] = 3,
    ["Four"] = 4
  }
```

Alternately it's worth considering if `()` would be the correct bracketing characters for such an expression.
