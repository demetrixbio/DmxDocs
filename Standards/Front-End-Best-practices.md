[[_TOC_]]

## Team practices for the front-end

As seen in the Front-end/Architecture document, we are following the Elmish (or MVU) architecture for building the apps. This pattern is simple and powerful but it still offers some flexibility when you need to deviate from the standard component layout to solve a problem. Here we will explain our architecture,  list those situations with recommendations on how to solve them in a uniform way across the team.

### Public face of a component

An Elmish component is usually kept in one file. For a clean API, the component should only expose the parts needed for interacting with it (normally Model/Message types and init/update/view functions). Because in F# module functions are public by default, it's easy to forget the `private` modifier.

```fsharp
module MyComponent

type Model = ...
type Msg = ...

// unit -> Model
let init = ...

// Msg -> Model -> Model * Cmd<Msg>
let update msg model = ...

// Model -> (Msg -> unit) -> ReactElement
let private subnavigation model dispatch = ...
let view model dispatch = ...
```

#### The view function

The view can become quite complicated in some components. Even though F# allows complicated expressions containing bindings, nested functions, etc, it's better to avoid this in the view functions, to make the hierarchy of the UI elements easier to see. Ideally the view function shouldn't contain much logic and be more declarative (closer to pure HTML), most of the logic should go in the update function.

> This doesn't mean the view function shouldn't contain any logic at all. One of the benefits of using Fable.Elmish is we can use the full power of the F# language to build our views, so we don't need complicated converters or templating systems to express dynamic behavior.

#### The init function

Besides the `update` function, all Elmish components include a `init` function. You can think of it as setting the initial state of your component. Because of this, its signature it's usually similar, but there are a couple of variations:

```fsharp
// default init function
let init: unit -> Model * Cmd<Msg>

// init function does not have to return a command
let init: unit -> Model

// accepts input from the parent
let init: ... -> Model * Cmd<Msg>
```

##### Asynchronous init: refactor this out of the apps and components!

It's better to avoid asynchronous `init` for a couple of reasons:

- In the Elmish architecture, asynchronous actions are dealt with using commands. F# `async` is somewhat a deviation of the Elmish way in that sense.
- Even if the component fetches its resources during page loading (when a loader is already displayed), it's possible that it has to do the same later and the component may look unresponsive then.

In general a component should be initialized quickly (synchronously) and return a command if it needs to perform an asynchronous operation. If data needs to be fetched, for example, the component should be initially displayed in a disabled state (and show a loader if necessary, so user knows something is happening in the background).

> Asynchronous init use to be the default for our applications (beginner's mistakes were made), but we started refactoring that out. It is possible you still find applications that use this.


### Communicating with the parent

The parent is unaware of the inner workings of its children, they are essentially black boxes. Sometimes a child component needs to communicate with its parent. In order to do that, the child creates a type `External messages` and we can make the `update` function return an additional external message. This is called the OutMsg pattern.

```fsharp
// CHILD COMPONENT
type ExternalMsg =
    | NoOp // Avoid shadowing Option.None
    | DoSomethingPlease

// Model -> Msg -> Model * Cmd<Msg> * ExternalMsg
let update msg model =
    match msg with
    | DoSomething ->
        let m = ...
        m, Cmd.none, NoOp
    | AskParentToDoSomething ->
        model, cmd, DoSomethingPlease

// PARENT UPDATE FUNCTION PATTERN MATCHES ON THE EXTERNAL MSG
let update msg model =
    match msg with
    | ChildMessage msg ->
        let child, cmd, extMsg = Child.update msg model.Child
        match extMsg with
        | Child.NoOp -> { model with Child = child }, Cmd.map ChildMessage cmd
        | Child.DoSomethingPlease -> // do what we're asked for
    ...
```

### Look and feel of components: use the builder pattern

Components need to look the same in all applications, but it is difficult to know what that look is. Our solution is to use the builder pattern. In this pattern, we create a module `MyComponentBuilder` which contains building blocks to create a reactelement in the look and feel of front-end. We include functions the consumers can use to instantiate and add properties to this object. For example:

```fsharp
[<RequireQualifiedAccess>]
module MyComponentBuilder

// This type is public but consumers don't instantiate it directly:
type Builder = { Width: int; Height: int }

let create() = { Width = 100; Height = 100 }

// Note the builder argument comes at the end so we can use F# pipe operator
let withWidth value builder = { config with Width = value }
let withHeight value builder = { config with Height = value }

let render (builder : Builder) = ...

// calling the Builder
MyComponentBuilder.create()
|> MyComponentBuilder.withWidth 300
|> MyComponentBuilder.withHeight 250
|> MyComponentBuilder.render
```

If we want to add an extra argument, we can just add a new function like `withDepth`. This will not affect already existing and will just use the default value for the new argument.

### Additional reading

* [Tips and tricks for Elmish components](https://medium.com/@MangelMaxime/my-tips-for-working-with-elmish-ab8d193d52fd)