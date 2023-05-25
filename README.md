# Elm Style Guide

_Reviewed last on 2023-05-24_

These are the styles and conventions that NoRedInk engineers use when writing Elm code. We've removed the guidelines that refer to internal scripts, so NoRedInk engineers should refer to the internal version of this document.

## Casing

Be exhaustive whenever possible in a case, in order catch unmatched patterns (and therefore bugs!) at compile time. This is especially important since it’s likely that new patterns will be added over time!

For example, do:

```elm
type Animal = Cat | Dog

animalToString animal =
  case animal of
    Dog -> "dog"
    Cat -> "cat"
```

Not:

```elm
type Animal = Cat | Dog

animalToString animal =
  case animal of
    Dog -> "dog"
    _ -> "cat"
```

## Identifiers

Don’t use simple types or type aliases for identifiers. Creating a custom type instead enables the compiler to find bugs (like passing a user id to a function that should take an assignment id) at compile time, not in production.

For example, do:

```elm
type StudentId = StudentId String
```

Not:

```elm
type alias StudentId = String
```

If you need a dict or set with `StudentId` as keys, use elm-sorter-experiment instead of the core dict and set implementations.

## Let bindings

Avoid giant `let` bindings by pulling functions out of `let`s to the top level of the file. This helps with readability and can make it more clear what is actually going on.

Another benefit is that by moving the functions to the top level, you are very likely to add type annotations. Type annotations aide in understanding how the code works!

Plus, pulling out functions forces you to declare all dependencies, rather than just relying on the scope. This can lead to refactoring insights — why does x function depend on y parameter, after all?

## Use anonymous function `\_ ->` over `always`

It's more concise, more recognizable as a function, and makes it easier to change your mind later and name the argument.

For example, do:

```elm
Maybe.map (\_ -> ())
```

Not:

```elm
Maybe.map (always ())
```

## Prefer parens to backwards function application `<|`

There are cases where it would be awkward to use parens instead of `<|` (for instance, in Elm tests), but in general, parens are preferable.

For example, do:

```elm
foo (bar (baz qux))
```

Not:

```elm
foo <| bar <| baz qux
```

## Avoid unnecessary forwards function application

If there’s only one step to a pipeline, is it even a pipeline? Save the forwards function application for when you’re using `andThen` (which can be confusing without it) or for when you’re doing multiple transformations.

For example, do:

```elm
List.map func list
```

Not:

```elm
list |> List.map func
```

## Always use descriptive naming (even if it means names get long!)

Clear, descriptive names of types, variables, and functions improve code readability. It’s much more important that code be readable than fast to type!

For example, do:

```elm
viewPromptAndPassagesAccordions : Model -> Html Msg
```

Not:

```elm
accdns : Model -> Html Msg
```

For a maybe value, do:

```elm
maybeUserId : Maybe UserId
```

Not:

```elm
userId_ : Maybe UserId
mUserId: Maybe UserId
```

(Note that in a generic function, like the internals of `Maybe.map`, a generic variable name `x` is descriptive!)

## Co-locate Flags and decoders

If you’re writing decoders by hand, co-locate the type you’re decoding into and its decoder.

Most decoders rely on the order of values in a record type alias in order to apply values against the type alias constructor in the right order. If a type alias and the decoder that uses it aren’t co-located, it makes it more difficult to ensure that the order of items in the type alias don’t change — which could cause bugs in production!

For example, do:

```elm
type alias User =
  { name : String
  , displayName : String
  , interests : List Interest
  }

userDecoder : Decoder User
userDecoder =
  succeed User
    |> required "name" string
    |> required "displayName" string
    |> required "interests" (list interestDecoder)

type alias Interest =
  { name : String
  }

interestDecoder : Decoder Interest
interestDecoder =
  succeed Interest
    |> required "name" string
```

Not:

```elm
type alias User =
  { name : String
  , displayName : String
  , interests : List Interest
  }

type alias Interest =
  { name : String
  }

userDecoder : Decoder User
userDecoder =
  succeed User
    |> required "name" string
    |> required "displayName" string
    |> required "interests" (list interestDecoder)

interestDecoder : Decoder Interest
interestDecoder =
  succeed Interest
    |> required "name" string
```

## Prefer explicit types in type signatures to type aliases

Elm compiler error messages are better when not using type aliases. Be aware of this when writing functions.

Consider doing:

```elm
func : { thing1 : String, thing2 : String } -> String
```

Rather than:

```elm
type alias FuncConfig =
  { thing1 : String
  , thing2 : String
  }

func : FuncConfig -> String
```

## Making impossible states impossible

Some states should never occur in your program. For example, there should never be two tooltips open at once! Depending on how you model your tooltip state, the type system might or might not prevent this bad state from occurring.

If you have a model with a list of assignments on it, and one assignment can have an open tooltip, your model should look like this:

```elm
type alias Model =
  { openTooltip = Maybe AssignmentId
  , assignments = List Assignment
  }
```

Not:

```elm
type alias Model =
  { assignments = List Assignment
  }

type alias Assignment =
  { openTooltip : Bool
  , ...
  }
```

Watch [Richard Feldman’s classic talk](https://www.youtube.com/watch?v=IcgmSRJHu_8) on this topic to learn more.

## Being strategic about making impossible states impossible

Making impossible states impossible is an awesome feature of a strong type system! However, it’s possible to go too far and make unlikely errors impossible at the expense of coding ergonomics and over-coupled code. This can make the codebase really hard to work with and change down the line. Sometimes ruling out one impossible state can also introduce other impossible states to the system.

Let’s take [our Guided Tutorials](https://www.noredink.com/t/797) as an example. They consist of a sequence of blocks that the user goes through. Blocks can also be grouped (visually represented as white containers), so once we are done with all blocks from a group we proceed to the next one. We may be tempted to model this as:

```elm
type alias Blocks = Zipper (Zipper Block)
```

Here, the top level zipper iterates through groups, and the inner zippers point to the current element within the group. This has some nice properties: we guarantee that containers are not empty and also also avoid (in principle) the need to handle `Maybe` values caused by having a separate `currentBlock` pointer. However, note that:

- This isn’t very ergonomic, as manipulating the data structure to advance the tutorial or modify a block’s state is really hard.
- In our attempt to rule out an impossible state we actually introduced multiple new ones! Each one of the inner zippers now has a different pointer to its current element, and there's an implicit invariant that all of the groups we already looked at will be fully advanced already. Breaking this invariant (ie. having a non-advanced zipper for one of the previous groups) represents an impossible state!

A better way to model this could be to use a grouping indicator at the moment of rendering the view in order to separate the blocks into containers:

```elm
-- NOTE: this is a simplified example, not exactly how tutorials code looks.
viewBlocks : Maybe BlockId -> List Block -> Html Msg
viewBlocks currentBlockId blocks =
  Html.div []
    (blocks
      |> List.Extra.groupBy .groupId
      |> List.map (viewContainer currentBlockId)
    )
```

This has the added benefit that our code will be more stable, in the sense that a the model isn't as coupled to the view as it was before. A whole class of changes to the view can now be implemented locally in the `view` code without having to change the fundamental data structure our whole program is built upon.

## Handling errors

It’s alright for things to go wrong in a program. It is going to happen! What’s important is that we show clear and specific error messages to the user when this does occur (and that we report some of these errors to our error-tracking service!).

## When to create a separate module

### Don’t break up modules based on the “shape” of things

Anti-patterns:

- `Foo.Model`
- `Foo.Msgs`
- `Foo.Update`
- `Foo.View`
- `Foo.Validation`

Watch [“The Life of a File”](https://www.youtube.com/watch?v=XpDsk374LDE) to learn about this concept in more depth.

Break up modules when you want to avoid invalid states
A good example is [`Percent`](https://github.com/NoRedInk/NoRedInk/blob/3caa7e9e24c7086687f687275e8d51dcb511f82a/monolith/ui/src/Percent.elm). Notice how it doesn’t expose a constructor for `Percent`, but exposes functions to create a `Percent`, i.e. `fromInt`.

If there was an exposed constructor for `Percent`, you could create a invalid percent `Percent (-1)`

### Adding extras to an already existing type Foo.Extra

Adding extras to a type can quickly get out of hand and it might become hard for future engineers to understand what a function does without constantly having to look up extras.
Generally it’s better to rely on already existing functions that are familiar to most engineers.
Only consider adding a new extra function if you’ve reused it in multiple places for multiple features. It’s okay to keep an extra function local to its usage until it has “proven” itself. Often such functions miss edge-cases or under go multiple naming iterations initially and it’s better to wait before moving it to a more general place.

### Extra2

We have some modules that add extras to module that are already open-sourced and established in the elm community i.e. `List.Extra` we have some additional extras in `List.Extra2`.
Whenever possible try to avoid this and instead contribute to the open-sourced version. This has the additional benefit of getting feedback from a broader audience.

## One Elm app per page

Avoid having multiple Elm entrypoints on a single page, as it messes up our ability to handle state well. For example, only 1 tooltip should be open at a time, but there’s no way to enforce this in the type system when there are multiple Elm apps running.