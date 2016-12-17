# Elm Style Guide

These are the guidelines we follow when writing [Elm](http://elm-lang.org) code at [NoRedInk](https://www.noredink.com/jobs).

Note to NoRedInkers: These conventions have evolved over time, so there will always be some parts of the code base that don't follow everything. This is how we want to write new code, but there's no urgency around changing old code to conform. Feel free to clean up old code if you like, but don't feel obliged.


## How to Name Modules

### `Nri`
`Nri.Button`, `Nri.Emoji`, `Nri.Leaderboard`

A reusable part of the site's look and feel, which could go in the visual style guide. While some parts could be made open source, these are tied directly to NRI stuff.

When adding a new abstraction to Nri, announce it on slack and seek as much feedback as possible! this will be used in multiple places.

#### Examples
- Common navigation header with configurable buttons

#### Non-examples
- `elm-css` colors and fonts should go in [here](https://github.com/NoRedInk/nri-elm-css)


### `Data`
`Data.Assignment`, `Data.Course`, `Data.User`

Data (and functions related to that data) shared across multiple pages.

#### Examples
- Data types for a concept shared between multiple views (e.g `Data.StudentTask`)
- A type that represents a "base" type record definition. A simple example might be a `Student`, which you will then extend in `Page` (see below)
- Helpers for those data types

### `Page`
`Page.Writing.Rate.Main`, `Page.Writing.Rate.Update`, `Page.Writing.Rate.Model.Decoder`

A page on the site, which has its own URL. These are not reusable, and implemented using a combination of types from `Data` and components from `Nri`.

Comments for usage instructions aren't required, as code isn't intended to be reusable.

#### Examples
- Entry points for our particular pages.


### Top-level modules
`Accordion`, `Dropdown`

Something reusable that we might open source, that aren't tied directly to any NRI stuff. Name it what we'd name it if we'd already open-sourced it.

Make as much of this opensource-ready as possible:

- Must have simple documentation explaining how to use the component. No need to go overboard, but it needs to be there. Imagine you're publishing the package on elm-package! Use `--warn` to get errors for missing documentation.
- Expose the Model, the Action constructors, the Address pattern.
- Use `type alias Model a = { a | b : c }` and `type alias Addresses a = { a | b : c }` to allow extending of things.
- Provide an API file as example usage of the module.
- Follow either the [elm-api-component](https://github.com/NoRedInk/elm-api-components) pattern, or the [elm-html-widgets](https://github.com/NoRedInk/elm-html-widgets) pattern

#### Examples
- Filter component
- Long polling component
- Tabs component


## Structure

Our Elm apps generally take this form:

- API.js.elm
- Model.elm
- Update.elm
- View.elm

API.js.elm is our entry file. Here, we import everything from the other files and actually connect everything together. We use a custom version of `StartApp`, that allows us to bootstrap our Elm program with data sent down from the server on load. We also setup the interop with JS in this file. Note the extension - API.js.elm. It has .js as it is the file that gets converted into JS! We run elm-make on this file to generate the component that we can include elsewhere.

Inside Model, we contain the actual model for the view state of our program. Sometimes, we include decoders for our model in here. Note that we generally don't include non-view state inside here, preferring to instead generalize things away from the view where possible. For example, we might have a record with a list of assignments in our Model file, but the assignment type itself would be in a general file called Assignment.elm.

Update.elm contains our update code. This includes the action types for our view. Inside here most of our business logic lives.

Inside View.elm, we define the view for our model and set up calls to any action that we need to.

To summarize:

- API.js.elm
    - Our entry point. Contains an initial model, startapp calls and ports.
    - Compile target for elm-make
    - Imports Model, Update and View.

- Model.elm
    - Contains the Model type for the view alone.
    - Imports nothing but generalized types that are used in the model

- Update.elm
    - Contains the Action type for the view, and the update function.
    - Contains `addresses`
    - Imports Model

- View.elm
    - Contains the view code
    - Imports Model and Update (for the action types)


## Tooling

### Use [`elm-init-scripts`](https://github.com/NoRedInk/elm-init-scripts) to start your projects

This will generate all the files mentioned above for you.

### Use [`elm-ops-tooling`](https://github.com/NoRedInk/elm-ops-tooling) to manage your projects

In particular, use `elm_deps_sync` to keep your main elm-package.json in sync with your test elm-package.json.

### Use [`elm-format`](https://github.com/avh4/elm-format) on all files

We run the latest version of [`elm-format`](https://github.com/avh4/elm-format) to get uniform syntax formatting on our source code.

This has [several benefits](https://github.com/avh4/elm-format#elm-format),
not the least of which is that it renders many potential style discussions moot,
making it easier to spend more time building things!

## Use descriptive names instead of apostrophes

Instead of this:

```elm
-- Don't do this --
markDirty model =
  let
    model' =
      { model | dirty = True }
  in
    model'
```

...just come up with a name.

```elm
-- Instead do this --
markDirty model =
  let
    dirtyModel =
      { model | dirty = True }
  in
    dirtyModel
```

## Use [`flip`](http://package.elm-lang.org/packages/elm-lang/core/2.1.0/Basics#flip) and/or pipes over backticks

Instead of this:

```elm
-- Don't do this --
saveAccounts (List.map deactivateAccount accounts)
  `andThen` (\response -> sendToLogger response.successMessage)
```

...use [`|>`](http://package.elm-lang.org/packages/elm-lang/core/2.1.0/Basics#|>) and qualified names like normal, and use `flip` to obtain the desired argument order.

```elm
-- Instead do this --
accounts
  |> List.map deactivateAccount
  |> saveAccounts
  |> (flip Task.andThen) (\response -> sendToLogger response.successMessage)
```

## Use [`\_ ->`](http://elm-lang.org/docs/syntax#functions) over [`always`](http://package.elm-lang.org/packages/elm-lang/core/2.1.0/Basics#always)

It's more concise, more recognizable as a function, and makes it easier to change your mind later and name the argument.

```elm
-- Don't do this --
on "click" Json.value (always (Signal.message address ()))
```

```elm
-- Instead do this --
on "click" Json.value (\_ -> Signal.message address ())
```

## Only use [`<|`](http://package.elm-lang.org/packages/elm-lang/core/2.1.0/Basics#<|) when parens would be awkward

Instead of this:

```elm
-- Don't do this --
foo <| bar <| baz qux
```

...prefer using parentheses, because they'd look fine:

```elm
-- Instead do this --
foo (bar (baz qux))
```

However this would be awkward:

```elm
-- Don't do this --
customDecoder string
  (\str ->
    case str of
      "one" ->
        Result.Ok 1

      "two" ->
        Result.Ok 2

      "three" ->
        Result.Ok 3
    )
```

...so prefer this instead:

```elm
-- Instead do this --
customDecoder string
  <| \str ->
      case str of
        "one" ->
          Result.Ok 1

        "two" ->
          Result.Ok 2

        "three" ->
          Result.Ok 3
```

## Always use [`Json.Decode.Pipeline`](https://github.com/NoRedInk/elm-decode-pipeline) instead of [`objectN`](http://package.elm-lang.org/packages/elm-lang/core/2.1.0/Json-Decode#object2)

Even though this would work...

```elm
-- Don't do this --
algoliaResult : Decoder AlgoliaResult
algoliaResult =
  object6 AlgoliaResult
    ("id" := int)
    ("name" := string)
    ("address" := string)
    ("city" := string)
    ("state" := string)
    ("zip" := string)
```

...it's inconsistent with the longer decoders, and must be refactored if we want to add more fields.

Instead do this from the start:

```elm
-- Instead do this --
import Json.Decode.Pipeline exposing (required, decode)

algoliaResult : Decoder AlgoliaResult
algoliaResult =
  decode AlgoliaResult
    |> required "id" int
    |> required "name" string
    |> required "address" string
    |> required "city" string
    |> required "state" string
    |> required "zip" string
```

This will also make it easier to add [optional fields](http://package.elm-lang.org/packages/NoRedInk/elm-decode-pipeline/1.0.0/Json-Decode-Pipeline#optional) where necessary.

[json2elm](http://json2elm.org/) can generate pipeline-style decoders from
raw JSON.

## Best practices

- All ports should bring things in as Json.Value. The single source of runtime errors that we have right now are through ports receiving values they shouldn't. If a `port something : Signal Int` receives a float, it will cause a runtime error. We can prevent this by just wrapping the incoming things as `Json.Value`, and handle the errorful data through a `Decoder` result instead.

- Ports should always have documentation. I don't want to have to go out from our Elm files to find where a port is being used most of the time. Simply adding a line or two explaining what the port triggers, or where the values coming in from a port can help a lot.

- Models for things that aren't tied to views shouldn't have _any_ view state within them. For example, an assignment should not have a `openPopout` attribute. Doing so means we can't use that type again in another situation.

- Use `case..of` over `if` where possible. It's clever as it will generate more efficent JS, and it aso allows you to catch unmatched patterns at compile time. It's also cheap to extend this data with something more useful later on, like if you need to add another branch. This saves code diffs.

- Keep the update function as small as possible. Smaller functions that don't live in a let binding are more reusable.

- If a function can be pulled outside of a let binding, then do it. Giant let bindings hurt readability and performance. The less nested a function, the less functions are used in generated code.

- Having complicated imports hurts our compile time! I don't know what to say about this other than if you feel that there's something wrong with the top 40 lines of your module because of imports, then it might be time to move things out into another module. Trust your gut.

- If your application has too many constructors for your Action type, consider refactoring. Large `case..of` statements hurts compile time. It might be possible that some of your constructors can be combined, for example `type Action = Open | Close` could actually be `type Action = SetOpenState Bool`
