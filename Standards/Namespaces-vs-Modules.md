This post attempts to summarize how code should be organized into namespaces and modules in general.

[[_TOC_]]

## Namespaces

On .NET platform (and not talking specifically about F# or C# here), everything is organized into namespaces.

Some key properties of namespaces:

1. Namespaces can be nested (there could be `Foo` , `Foo.Bar` and also `Foo.Bar.Baz`
2. Namespaces can contain **types** and cannot contain **values**

What does that mean? You can create a new `class Foo` (C#) or new `type Foo` in a namespace but you cannot have an instance of them bound to a variable:

```csharp
namespace Foo {

    string bar = "baz" // this won't compile
}
```

```fsharp
namespace Foo

let bar = "baz" // this also won't
```

3. A fully qualified name of a type is the parent namespace + the name of the type:

```fsharp
namespace Foo.Bar

type Baz = { Is: bool }
```

The fully qualified name of `Baz` is `Foo.Bar.Baz`

4. Something cannot be both a namespace and a type

If you have a type `Bar` in `Foo` namespace (`Foo.Bar`), then you can no longer define a type with a fully qualified name as `Foo.Bar.Baz`.

It is forbidden, because the container namespace of `Baz` would be `Foo.Bar` but that's already a type.

(this has some serious consequences in F#, defining a module will prevent the namespace hierarchy to expand further down below a module. More on that later)

5. You can achieve "namespacing" inside types by using nested types but that is a very limited thing compared to  real namespaces:

```csharp
namespace Foo {
    class Bar {
        class Baz {
        }
    }
}
```

In this case the fully qualified name of `Baz` is `Foo.Bar.Baz`. The shortest name that you can refer to `Baz` is `Bar.Baz` from then.

## Modules in F#

So what are modules in F#? Although they might look like *namespaces* first, or something like a mixture of *types* and *namespaces*, **they are types** with some syntactic sugar support from the compiler.

They are compiled to a very similar thing to a `static class` in C#:

```fsharp
namespace Foo

module Bar =
  let baz () = "hello"
```

is roughly equivalent of:

```csharp
namespace Foo {
    static class Bar {
        public static String baz() {
            return "hello";
        }
    }
}
```

Usage in another file:
```fsharp
open Foo

printfn "%s" (Bar.baz()) // "hello"
```

### Can be `open`-ed

Why do they have simliarity with namespaces? Because *they can be `open`-ed*:
```fsharp
open Foo.Bar // open Foo -> open Foo.Bar

printfn "%s" (baz()) // Bar.baz() -> baz()
```

If you `open` a module, it will make all of its content be available **without any prefix**. (= you no longer have to write `Bar.`)

This way F# does some cheating because if you define a type `A` inside a module `B`, and open the module `B`, you can refer to that type `A` without the module prefix `B.`, while you cannot do this in C#. Let's reconsider property #5 from the namespaces section:

```csharp
namespace Foo {
    class Bar {
        class Baz {
        }
    }
}
```

In C#, no matter what you do, you can refer to `Baz` only at least as `Bar.Baz`. Period.

In F#, you can say this:
```fsharp
namespace Foo

module Bar =
  type Baz = { ... }
```

in another file normally:
```fsharp
open Foo

let x: Bar.Baz = { ... }
```

in another file, opening the module:
```fsharp
open Foo.Bar

let x: Baz = { ... }
```

### Can contain values!

It acts like a namespace in some cases, but it could be more powerful than namespaces as namespaces cannot contain values while modules can:

```fsharp
module Bar

let bar = "baz"
```

If you use `Bar` as a namespace as described above, you can "add values to namespaces" (define global constants, add functions to namespaces). Very important to emphasize, while it looks like you are adding these to a namespace, you are actually adding it to a type! (to a static class) And it is the F# compiler that makes you feel like you have added it to a namespace.

### Modules with the same name are merged

If two modules of two different namespaces have the same name and both namespaces are opened in a third file, the F# compiler will merge those modules which leads to a very interesting and useful construct:

```fsharp
file1:
namespace Domain

type Bar = { Field: string }

module Bar =
  let getValue (this: Bar): string = this.Field

file2:
namespace Domain.Serialization

module Bar =
  let toJson (this: Bar): JsonValue = Json.serialize this

file3:
open Domain
open Domain.Serialization
let myValue = { Bar.Field = "hello" }

printfn "%s" (myValue |> Bar.getValue) // "hello"
printfn "%A" (myValue |> Bar.toJson)  // "{ "Field": "hello" }"
```

It might not be immediately apparent but this is something like capturing different aspects of a given type in different files. Just imagine file1 contains the core definition of `Bar` along with some domain logic, while file2 contains some kind of serialization logic for the same type. If a caller code is interested only in the serialization aspect, it opens only `Domain.Serialization` and doesn't pollute the coding environment with domain specific logic related symbols. However, if a user needs both domain logic and serialization logic then it can open both namespaces and can access both aspects under the same name: `Bar`

#### What happens with colliding names?

What happens if both `Domain` and `Domain.Serialization` define a function called `foo` ?

-the namespace you opened later wins. In the above case, in `file3`, `Domain.Serialization.Bar.foo` would be available.

### Caveats

Like many other things `modules` look very powerful and indeed they are, but they are a double edged sword.

#### Polluted coding environment

It is a very common mistake to put a type and its corresponding operations inside a module (therefore using it instead of a namespace):

```fsharp
module Foo.Domain

type Person = { Name: string }


let getName (this: Person) = this.Name
```

we are trying to use it from another file:
```fsharp
open Foo.Domain

let myPerson = { Name = "Jack" }

printfn "%s" (myPerson |> getName) // "Jack"
```

Although this looks nice and is very good in such a simple case, let's consider a lot more complex scenario where 5 more modules are imported using open, each of them contain a lot of types and functions with the same pattern, without prefixing using any kind of further qualifier. How do you tell where a given `foo()` function belongs to? Especially if you are not looking at the code from an IDE. Even if you prefix `getName` with `Person` then the IDE would start complaining that `Person` is a redundant qualifier here. It is indeed redundant for the compiler but it is not about whether the compiler knows where `getName` belongs to, it's rather the reader of the code. The human doesn't know unless knows every piece of code around here. Not to mention the case where we import `Pet` which also has a `getName`. Then you have to start aliasing and such.

#### No function overloading

We discussed that modules look like namespaces but they are not really namespaces. They look like static classes and it turns out they are not really static classes either.

There is one feature missing compared to static classes: function overloading. You cannot overload a function in a module:

```
module Foo

let bar (input: string) = ...
let bar (input: int) = ... // this won't compile!
```

This is because each name inside a module must be unique therefore having two functions with the same name but different signature is forbidden.

Fortunately, **you can create real static classes** in F#:

```fsharp
[<AbstractClass; Sealed>]
type Foo private () =
    static member Bar(a: string) =
       ...

    static member Bar(b: int) =
       ...
```

This is very useful when creating libraries.

## Best practices

Now that we have seen the pros and cons, how code should be organized?

It is always a good practice to follow what the `FSharp.Core` library does in the F# compiler:

1. Place the `type`s to **namespaces**
2. Add the associated functionality to a **module** with the same name as the type
3. You can split the functionality based on different aspects to different namespaces but keep the **same module name**
4. In caller code, open the container namespaces, therefore refer to the type `X` as `X` and the associated fuctions qualified with the module name `X.foo`
5. Exception: it is totally acceptable to create a type inside a module in case that type is used exclusively by the functions of the module. You should mark this type as `private`. Also you can make it `internal` in order to make it visible to unit tests, but ensure the type is not used by other modules

### Example 1 (Person)

Let's say you want to have a Person like thing around the Warehouse namespace. Person supports some domain functionality like `getName`, `setAge` and some serialization functionality: `toJson`

Then Person.fs looks like this:
```fsharp
namespace Warehouse

type Person =
  { Name: string
    Age: int }

module Person =
  let getName (this: Person): string = this.Name

  let setAge (age: int) (this: Person): Person = { this with Age = age }
```

this is analoguous with the object oriented style:

```csharp
namespace Warehouse {
  class Person {
    private String name;
    private int age;

    public String Name { get { return this.name; } }
    public void setAge (int newAge) {
        this.age = newAge;
    }
  }
}
```

and Person.Serialization.fs like this:

```fsharp
namespace Warehouse.Serialization
open Warehouse
open Newtonsoft.Json

module Person =
  let toJson (this: Person): JsonValue = this |> Json.serialize
```
which could be an extension method in C#:
```csharp
namespace Warehouse.Serialization {
  static class PersonExtensions {
    public JsonValue toJson (this: Person) {
      return Json.serialize(this);
    }
  }
}
```
(the same could be achieved with having Person as a Partial class)

An example usage of a subset of person:
```fsharp
open Warehouse.Serialization

let myPerson = PersonService.queryByName "John"

let serialized = myPerson |> Person.toJson

printfn "%A" serialized
```

Please note, i did not need the full domain specific aspect of Person since i am only serializing it here!

### Example 2 (FSharp.Core.Collections)

Just a quick example of how things are defined in the core F# library:

```fsharp
namespace Microsoft.FSharp.Collections

type List = :: | []

module List =
  let head (this: List) = ...

```