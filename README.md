These are the guidelines we follow when writing [Elm](http://elm-lang.org) code at [NoRedInk](https://www.noredink.com/jobs).

## Structure

Our Elm apps generally take this form:

- API.js.elm
- Model.elm
- Update.elm
- View.elm

API.js.elm is our entry file. Here, we import everything from the other files and actually connect everything together. We use a custom version of StartApp, that allows us to bootstrap our Elm program with data sent down from the server on load. We also setup the interop with JS in this file. Note the extension - API.js.elm. It has .js as it is the file that gets converted into JS! We run elm-make on this file to generate the component that we can include elsewhere. Inside Model, we contain the actual model for the view state of our program. Sometimes, we include decoders for our model in here. Note that we generally don't include non-view state inside here, preferring to instead generalize things away from the view where possible. For example, we might have a record with a list of assignments in our Model file, but the assignment type itself would be in a general file called Assignment.elm. Update.elm contains our update code. This includes the action types for our view. Inside here most of our business logic lives. Inside View.elm, we define the view for our model and set up calls to any action that we need to.

To summarize:

- API.js.elm
    - Our entry point. Contains an initial model, startapp calls and ports.
    - Compile target for elm-make
    - Imports Model, Update and View.

- Model.elm
    - Contains the Model type for the view alone. 
    - Imports nothing but generalized types that are used in the model

- Update.elm
    - Contains the Action type for the view, and the function. 
    - Imports Model

- View.elm
    - Contains the view code
    - Imports Model and Update (for the action types)

A good way of starting your projects is to use [elm-init-scripts](https://github.com/NoRedInk/elm-init-scripts) which will generate these files for you.

## Use [`elm-format`](https://github.com/avh4/elm-format) on all files

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
