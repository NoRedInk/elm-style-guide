These are the guidelines we follow when writing [Elm](http://elm-lang.org) code at [NoRedInk](https://www.noredink.com/jobs).

Use elm-format.

We start with the official [Elm Style Guide](http://elm-lang.org/docs/style-guide) and then add the following.

## Always put [`if`](http://elm-lang.org/docs/syntax#conditionals) and [`then`](http://elm-lang.org/docs/syntax#conditionals) on the same line

Instead of this:

```elm
-- Don't do this --
if isOdd number
    then Odd
    else
        if isEven number
            then Even
            else WhoaCrazyNumber
```

...put `then` on the same line, and indent `else` the same as `if`:

```elm
-- Instead do this --
if isOdd number then
    Odd
else if isEven number then
    Even
else
    WhoaCrazyNumber
```

## Updating fields: Put `{ model ` on its own line and indent the fields being updated

Instead of this:

```elm
UpdateInviteCode str ->
    { model | inviteCode = str }

ReenterInviteCode ->
    { model | inviteCode = ""
    , formStep = WaitingForInviteCode
    }
```

Do this:

```elm
UpdateInviteCode str ->
    { model
         | inviteCode = str
    }

ReenterInviteCode ->
    { model
         | inviteCode = ""
         , formStep = WaitingForInviteCode
    }
```

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

## Always use [`|:`](https://github.com/circuithub/elm-json-extra/blob/master/src/Json/Decode/Extra.elm#L90-L113) instead of [`objectN`](http://package.elm-lang.org/packages/elm-lang/core/2.1.0/Json-Decode#object2)

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
import Util.Decoder.Extra exposing ((|:))

algoliaResult : Decoder AlgoliaResult
algoliaResult =
    succeed AlgoliaResult
        |: ("id" := int)
        |: ("name" := string)
        |: ("address" := string)
        |: ("city" := string)
        |: ("state" := string)
        |: ("zip" := string)
```

where `|:` is defined as

```elm
import Json.Decode.Extra
(|:) = Json.Decode.Extra.apply
```