---
author: Matthías Páll Gissurarson
date-accepted: ""
proposal-number: ""
ticket-url: ""
implemented: ""
---

This proposal is [discussed at this pull request](https://github.com/ghc-proposals/ghc-proposals/pull/0>).
**After creating the pull request, edit this file again, update the number in
the link, and delete this bold sentence.**

# Extended Typed-Holes

Typed-holes are a powerful way for users to interact with the compiler during
compilation, to ask for more information about the context and (recently) to
get suggestions on what could be used in place of the hole. However, the user
can only influence the name of the hole, with the rest being determined by the
hole's context.  We propose to add a new extension, `ExtendedTypedHoles` to
allow users to communicate more efficiently with the compiler by adding 3 new
syntactic constructs to GHC that represent typed-holes by using `_(...)`,
`_$(...)` and `_$$(...)` where `...` is an expression, a template haskell
expression or a typed template haskell expression respectively.


## Motivation

Typed-holes have a lot of potential for users to really interact with the
compiler during compilation beyond just parsing and type errors. Currently,
typed hole plugins allow plugin developers to extend how these holes are
handled. But, the only way the user can pass any information to the plugin is
via flags or via the name of the hole, which is not an ideal situation.
This means that all the information must be passed along as alphanumeric
strings (without spaces in case of names!), and not in the type safe manner
of Haskell data structures as we'd like. Strings do allow us to define some
language that the user would have to learn, but does not allow for expressive
combinators.

This limits the usefulness of typed-hole plugins and means that IDE features
that interact with holes would either need to pass complex flags or hard to
read names to plugins for any advanced functionality such as limiting synthesis
to specific modules or constraints. 

As an example, consider the [DjinnHoogleModPlugin](https://github.com/Tritlo/ExampleHolePlugin/tree/master/djinn-hoogle-mod-plugin):
```
{-# OPTIONS -fplugin=DjinnHoogleModPlugin
            -funclutter-valid-hole-fits #-}
module Main where
import Control.Monad
f :: (a,b) -> a
f = _invoke_Djinn
g :: [a] -> [[a]]
g = _invoke_Hoogle
h :: [[a]] -> [a]
h = _module_Control_Monad


main :: IO ()
main = return ()
```

Here, the name of the hole is used to invoke different utilities and filter by
specific modules, but the limitation of having only the name makes it impractical
to invoke more than one command at a time, and requires users to use `_` in the
names of the modules we want to filter by.


## Proposed Change Specification

We add `-XExtendedTypedHoles` to the available extensions, which turns
on the lexing of the following tokens:
+ `_(` for opening an extended typed-hole, whose content is either
  nothing, or a haskell expression.
+ `\) $idchar $idchar*` for allowing users to enumerate the holes, with
   e.g. `_(...)0`, ..., `_(...)n`, or name them e.g. `_(...)a`
   or `_(...)fix_this`.
+ `_$(` opens a extended typed-hole containing a template haskell expression
  inside.
+ `_$$(` opens an extended typed-hole containing a typed template haskell
  expression.

Both `_$(` and `_$$(` require `TemplateHaskell` to be enabled in addition to
`ExtendedTypedHoles`.

We extend the grammar by adding the `extended_typed_hole` construct to `aexp2`
and `hole_op`, where `extended_typed_hole` is:

```
extended_typed_hole
        : '_('       hole_close
        | '_('   exp hole_close
        | '_$('  exp hole_close
        | '_$$(' exp hole_close

hole_close
  : ')'
  | CLOSE_HOLE

```

where `CLOSE_HOLE` is the `\) $idchar $idchar*` lexeme.

These are parsed into a new `HsExpr`, `HsExtendedHole`, contains the
`LHsExpr` from the hole or a template haskell splice.

For the splices, they behave the same as template haskell splices in that
the untyped splice is run during renaming, and the typed template haskell
splice is run during  type-checking, and generally behave in the same way 
as regular template haskell splices, i.e. they are run at the same time.
The only difference is that the resulting expression is not spliced into
the code, but rather passed along with the `ExtendedExprHole` to the
constraint solver.

The resulting expressions are then wrapped in a `toDyn` call from 
`Data.Dynamic` prior to being type-checked, and the
`runExtHExpr :: LHsExpr GhcTc -> TcM Dynamic` function is provided
in `TcHoleErrors`. This allows plugin developers to easily run the
expressions and obtain a `Dynamic`, which they can then safely try to
interpet as whatever type they expect the user to use in the holes.

When the type-checker encounters a `HsExtendedHole` expression, it emits
an insoluable `CHoleCan` (as we do for typed-holes already), but one that
contains an `ExtendedExprHole` constraint with the expression (if any) or
the expression resulting from running the splice, as well as the inferred
type of that expression. The expression can then be accessed via the
constraint from plugins or the error reporter, and will hopefully enable
us to move towards [Extensible Type-Directed Editing](http://cattheory.com/extensibleTypeDirectedEditing.pdf)
in Haskell.

As it is behind an extension flag, it does not impact any existing parsing
nor features, except Haddock and other Haskell AST parsing utilities will 
need to handle the new `HsExpr`.

## Examples

As an example, consider the [ExtendedHolesPlugin](https://github.com/Tritlo/ExampleHolePlugin/tree/master/extended-holes-plugin).
By defining a DSL, the plugin can express how users can interact with it
via template haskell expressions.


This allows us to compile the following:

```
{-# OPTIONS -fplugin=ExtendedHolesPlugin -funclutter-valid-hole-fits #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE ExtendedTypedHoles #-}
module Main where
import ExtendedHolesPlugin
import Control.Monad

f :: (a,b) -> a
f = _([Hoogle])


g :: (a,b) -> b
g = _$( exec $ do
        invoke Hoogle
        filterBy "Control.Monad"
        invoke Djinn)0

h :: (a,b) -> b
h = _$$( execTyped $ do
         filterBy "Prelude"
         invoke Djinn)1

```

where `invoke :: PluginType -> Cmd ()`, `filterBy :: String -> Cmd ()`,
`exec :: Cmd () -> Q Expr`, and the `execTyped :: Cmd () -> Q (TExp [PluginType])`
functions are defined by the plugin itself. This results in the following output:

```
Main.hs:9:5: error:
    • Found hole: _(...) :: (a, b) -> a
      Where: ‘b’, ‘a’ are rigid type variables bound by
               the type signature for:
                 f :: forall a b. (a, b) -> a
               at Main.hs:8:1-15
      Or perhaps ‘_(...)’ is mis-spelled, or not in scope
    • In the expression: _(...)
      In an equation for ‘f’: f = _(...)
    • Relevant bindings include f :: (a, b) -> a (bound at Main.hs:9:1)
      Valid hole fits include
        Hoogle: Prelude fst :: (a, b) -> a
        Hoogle: Data.Tuple fst :: (a, b) -> a
        f :: (a, b) -> a
        fst :: forall a b. (a, b) -> a
  |
9 | f = _([Hoogle])
  |     ^^^^^^^^^^^

Main.hs:13:5: error:
    • Found hole: _$(...)0 :: (a, b) -> b
      Where: ‘a’, ‘b’ are rigid type variables bound by
               the type signature for:
                 g :: forall a b. (a, b) -> b
               at Main.hs:12:1-15
      Or perhaps ‘_$(...)0’ is mis-spelled, or not in scope
    • In the expression: _$(...)0
      In an equation for ‘g’: g = _$(...)0
    • Relevant bindings include
        g :: (a, b) -> b (bound at Main.hs:13:1)
      Valid hole fits include
        (\ (_, a) -> a)
        Hoogle: Prelude fst :: (a, b) -> a
        Hoogle: Data.Tuple fst :: (a, b) -> a
        g :: (a, b) -> b
   |
13 | g = _$( exec $ do
   |     ^^^^^^^^^^^^^...

Main.hs:19:5: error:
    • Found hole: _$$(...)1 :: (a, b) -> b
      Where: ‘a’, ‘b’ are rigid type variables bound by
               the type signature for:
                 h :: forall a b. (a, b) -> b
               at Main.hs:18:1-15
      Or perhaps ‘_$$(...)1’ is mis-spelled, or not in scope
    • In the expression: _$$(...)1
      In an equation for ‘h’: h = _$$(...)1
    • Relevant bindings include
        h :: (a, b) -> b (bound at Main.hs:19:1)
      Valid hole fits include
        (\ (_, a) -> a)
        (\ (_, a) -> g (head (cycle (([]) ++ ([]))), a))
        h :: (a, b) -> b
        g :: forall a b. (a, b) -> b
        snd :: forall a b. (a, b) -> b
   |
19 | h = _$$( execTyped $ do
   |     ^^^^^^^^^^^^^^^^^^^...
```

## Effect and Interactions

By being able to parse any string and the result of template haskell expressions
along to the typed-hole plugins, we enable much richer interaction that the user
can define during development.

## Costs and Drawbacks

As it is mostly a change in how we parse and pass along information, the cost
is not high from GHC's point of view. Further work will be needed from any 
typed hole plugins to make effective use of this new information, but since
this is behind an extension and does not interfere with previous functionality,
this cost is minor.

The extended typed-holes are a bit notation heavy, but since they always result in
a type error, complete programs are unlikely to have holes in them.

## Alternatives
+ Do nothing, and use the names of holes as a way of communicating with plugins.

## Unresolved Questions

+ Is `_(...)` the right syntax? Other alternatives could be `_{...}` to mirror
  Agda, or even something entirely different like `<...>`. `_(...)` and `_$(...)`
  matches the current template haskell syntax nicely though, and it follows the
  convention that things on the right hand side that start with an underscore
  are typed-holes.
+ Is `_(...)idchars` a form that is neccessary, or is it enough that users can
  use `_(...)0`, ..., `_(...)n` to disambiguate between holes?
+ Should we make `ExtendedTypedHoles` and the `_(...)`  syntax available
  without requiring the extension flag enabled? This would allow us to
  use `_` for regular typed-holes (without any suggestions etc.), and require users
  that want suggestions or to use plugins to use  the `_(...)` syntax.
+ Should we define and require that the template haskell splices have a specific
  type, and do away with `Dynamic`?  This could ease the interop between different
  plugins, but we'd also like to give plugin developers as much freedom as possible.
+ Should we allow the hole plugins to solve the constraint and result in a splice
  instead of an error? This seems like an avenue that could be explored further in
  the future.

## Implementation Plan

The implementation will be done by @Tritlo, and a prototype is available at [!1766](https://gitlab.haskell.org/ghc/ghc/merge_requests/1766).