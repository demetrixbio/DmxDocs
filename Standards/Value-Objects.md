[[_TOC_]]

### What is a value object?

##### The short version:

Value objects are types that have no identity because they don't need to be uniquely identifiable inside your system. You care about the value it has, not it's identity.
(When you care about it's identity, it is called an entity)

##### The original version:

> An object that represents a descriptive aspect of the domain with `no conceptual identity` is called a `Value Object`. Value Objects are instantiated to represent elements of the design that `we care about only for what they are`, not who or which they are.
> ~ Eric Evans

### Characteristics

- No identity
- Immutable
- They don't exist without "a parent" (often referred to as entity)
- Often represent columns in a table (the entity is the table itself)

### How do I create one in the code?

In the codebase we use SCU's (single case union) for this. Here is an example:

``` fsharp
[<Struct>]
type TagName = TagName of name : string
with
    member this.Value with get() = let (TagName value) = this in value
module TagName =
    let get (TagName id) = id

```

They can become a bit more complicated sometimes. Here is an example of that:

``` fsharp
[<Struct>]
type ExperimentWorkflowId =
    { TypeName : TagName
      TypeId : TagIdentifier
      Ordinal : StageOrdinal }
with
    member x.Label = sprintf "%s%i" x.TypeName.Value x.Ordinal.Value
    override x.ToString() = x.Label
    static member GetLabel (id : ExperimentWorkflowId) = id.Label
  
```

When they are shared between client and server, we store them in the `ValueObject.fs`, when they are only used in the back-end, we store them in the Data project, close to the Storage is belongs to.

### You said they make no sense without an entity, but I see them being used without being part of an entity!

That is a very good observation. That is because they also offer us technical benefits, such as strong typing . No more bugs because you swapped to parameters!
Compare following functions:

``` fsharp
// #%&$, I swapped cropId and entryVector. I am such an idiot.
let checkCrop (userId : int) (cropId : int) (entryVector : int) =
    ...

// yeey, no more feeling stupid!
let checkCrop (requestedBy : UserIdentifier) (crop : CropIdentifier) (entryVector : PartIdentifier) =
    ...

```

They also make the code more expressive, f.e. we now know that entry vector is a part, because the function expects a `PartIdentifier`.

### Uhu, I still don't get it:

We understand, it takes some getting use to. Just think about the role the object has inside the domain.
Until you get the hang of it, here are some heuristics to work with:

> does it express something that needs to be `uniquely` identifiable?
> - YES! -> It is not a value object
> - NO! -> It is probably a value object
>
> is it a table or a column inside a table?
> - A table -> It is not a value object
> - A column -> It is probably a value object

### Resources

Other people might be better at explaining this to you, so we added some resources!

- [Value objects and other concepts of DDD](https://www.domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf)