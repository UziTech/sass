# `@extend`

The `@extend` at-rule is probably the single most complicated feature in Sass.
While its semantics are straightforward to describe, the implementation involves
many interacting layers and a lot of intricate case analysis.

## Table of Contents

* [Definitions](#definitions)
  * [Extender](#extender)
  * [Target](#target)
  * [Extension](#extension)
  * [Extendee](#extendee)
  * [The `extend()` Function](#the-extend-function)
* [Semantics](#semantics)
  * [Executing an `@extend` Rule](#executing-an-extend-rule)
  * [Resolving a Module's Extensions](#resolving-a-modules-extensions)
  * [Limitations](#limitations)
  * [Specificity](#specificity)
    * [The First Law](#the-first-law)
    * [The Second Law](#the-second-law)

## Definitions

These definitions provide names to the various selectors involved with a given
use of `@extend`:

```scss
.extender {
  @extend .target;
}

// ┌─ extendee
.target {
  // ...
}
```

### Extender

An `@extend` rule's *extender* is the [selector list][] for the style rule in
which the `@extend` rule appears.

[selector list]: https://drafts.csswg.org/selectors-4/#selector-list

### Target

An `@extend` rule's *target* is the [simple selector][] that's used as an
argument to `@extend`.

[simple selector]: https://drafts.csswg.org/selectors-4/#simple

### Extension

An *extension* is a collection of various properties.

> An extension is a more abstract representation of the information inherent in
> an `@extend` rule. As such, all `@extend` rules define extensions, but not all
> extensions directly correspond to `@extend` rules.

* The *extender*, a [selector list][].
* The *target*, a [simple selector][].

### Extendee

An *extendee* is a selector list being modified by an [extension](#extension).
It's only defined within the scope of a single application of a given extension.

> If an extendee contains that extensions's target, it will usually be modified
> to include the extension's extender as well.

### The `extend()` Function

As a shorthand, we use the function notation `extend(extendee, target,
extender)` to refer to the result of extending `extendee` with `extender
{@extend target}` (much like the Sass function `selector-extend()`). We further
use `extend(extendee, extension)` as a shorthand for `extend(extendee,
extension.target, extension.extender)`.

## Semantics

The `@extend` rule means that all elements matching the [extender](#extender)
should be styled as though they match the [target](#target) as well. The
`@extend` rule only applies to CSS in the module in which it's defined and
that module's transitive dependencies.

> Because Sass can't directly affect how the browser applies styles to elements,
> these semantics are approximated by duplicating each [extendee](#extendee)
> with the target replaced by the extender. Rather than being a naïve textual
> replacement, the extender is integrated intelligently into the extendee to
> match the semantics as best as possible.

### Executing an `@extend` Rule

To execute an `@extend` rule `rule`:

* If there is no [current style rule][], throw an error.

  [current style rule]: ../style-rules.md#current-style-rule

* Let `selector` be the result of evaluating all interpolation in `rule`'s
  selector and parsing the result as a list of simple selectors.

* If `selector` contains any parent selectors, throw an error.

* Let `extension` be an [extension](#extension) whose extender is the current
  style rule's selector and whose target is `selector`.

* Add `extension` to [the current module][]'s extensions.

  [the current module]: ../spec.md#the-current-module

> Note that this adds the extension to the module being evaluated, not the
> module in which the `@extend` lexically appears. This means that `@extend`s
> are effectively dynamically scoped, not lexically scoped.

### Resolving a Module's Extensions

This algorithm takes a [module][] `starting-module` and returns a [CSS tree][]
that includes CSS for *all* modules transitively used or forwarded by
`starting-module`.

[module]: ../modules.md#module
[CSS tree]: ../modules.md#css-tree

* Let `new-selectors` be an empty map from style rules to selectors. For the
  purposes of this map, style rules are compared using *reference equality*,
  meaning that style rules at different points in the CSS tree are always
  considered different even if their contents are the same.

* Let `new-extensions` be an empty map from modules to sets of
  [extensions](#extension).

* Let `extended` be the subgraph of the [module graph][] containing
  modules that are transitively reachable from `starting-module`.

  [module graph]: ../modules.md#module-graph

* For each module `domestic` in `extended`, in reverse [topological][] order:

  [topological]: https://en.wikipedia.org/wiki/Topological_sorting

  * Let `downstream` be the set of modules in `extended` whose dependencies
    include `domestic`.

  * For each style rule `rule` in `domestic`'s CSS:

    * Let `selector` be the result of applying `domestic`'s extensions to
      `rule`'s selector.

    * Let `selector-lists` be an empty set of selector lists.

    * For each module `foreign` in `downstream`:

      * Let `extended-selector` be `extend(selector, new-extensions[foreign])`.

        > `new-extensions[foreign]` is guaranteed to be populated at this point
        > because `extended` is traversed in reverse topological order, which
        > means that `foreign`'s own extensions will already have been resolved
        > by the time we start working on modules upstream of it.

      * Add `selector` to `selector-lists`.

    * Set `new-selectors[rule]` to a selector that matches the union of all
      elements matched by selectors in `selector-lists`. This selector must obey
      [the specificity laws](#specificity) relative to the selectors from which
      it was generated. For the purposes of [the first law](#the-first-law),
      "the original extendee" is considered only to refer to selectors that
      appear in `domestic`'s CSS, *not* selectors that were added by other
      modules' extensions.

      > Implementations are expected to trim redundant selectors from
      > `selector-lists` as much as possible. For the purposes of the first law
      > of extend, "the original extendee" is *only* the selectors in `rule`'s
      > selector. The new complex selectors in `selector` generated from
      > `domestic`'s extensions don't count as "original", and may be optimized
      > away.

    * For every extension `extension` whose extender appears in `rule`'s
      selector:

      * For every complex selector `complex` in `new-selectors[rule]`:

        * Add a copy of `extension` with its extender replaced by `complex` to
          `new-extensions[domestic]`.

* Let `css` be an empty CSS tree.

* Define a mutating recursive procedure, *traversing*, which takes a module
  `domestic`:

  * If `domestic` has already been traversed, do nothing.

  * Otherwise, traverse every module in `domestic`'s dependencies.

    > Because this traverses modules depth-first, it emits CSS in reverse
    > topological order.
    
  * Let `initial-imports` be the longest initial subsequence of top-level
    statements in `domestic`'s CSS tree that contains only comments and
    `@import` rules *and* that ends with an `@import` rule.

  * Insert a copy of `initial-imports` in `css` after the last `@import` rule, or
    at the beginning of `css` if it doesn't contain any `@import` rules.

  * For each top-level statement `statement` in `domestic`'s CSS tree after
    `initial-imports`:

    * If `statement` is an `@import` rule, insert a copy of `statement` in `css`
      after the last `@import` rule, or at the beginning of `css` if it doesn't
      contain any `@import` rules.

    * Otherwise, add a copy of `statement` to the end of `css`, with any style
      rules' selectors replaced with the corresponding selectors in
      `new-selectors`.

* Return `css`.

### Limitations

It's not possible for a preprocessor to guarantee the semantics of `@extend` in
full generality. There are three major exceptions where implementations are not
required to meet the full definition.

1. Implementations should not try to apply native browser styles that would
   apply to the target. For example, while it's legal to write `@extend table`,
   there's no good way to apply browsers' built-in table styles.

2. Second, when the extender and the extendee both contain multiple compound
   selectors separated by combinators, implementations are allowed to assume
   that the elements matched by the extender's compound selectors are not
   interleaved with those matched by the extendee's compound selectors.

   For example, consider `extend(.c .x, .x, .a .b)`. Implementations must
   generate the selectors `.a .c .b` and `.c .a .b`, because an element with
   `class="a"` may be either outside or inside one with `class="c"`. However,
   implementations are not required to generate the selector `.a.c .b` which
   would require HTML with `class="a c"`.

   This flexiblity is allowed because otherwise implementations would have to
   generate a combinatorial explosion of selectors, the vast majority of which
   would be extremely unlikely to match real HTML. This particular heuristic
   assumes that the extender and extendee were each written with self-contained
   HTML in mind, so that interwoven HTML is unlikely to come up.

3. Implementations are not required to apply the target's styles with the exact
   same specificity as the extender, because this isn't generally possible when
   complex extendees exist. However, implementations must respect certain
   guarantees about specificity; see below for details.

### Specificity

When modifying the extendee during extension, the implementation must provide
two guarantees about the result. These are known as the "laws of extend".

#### The First Law

The first law of `@extend` says that the specificity of the first generated
selector must be greater than or equal to that of the original extendee. For
example, `extend(a.foo, .foo, .a)` should generate `a.foo, a` even though
`a.foo` matches a subset of elements matched by `a`.

In most cases, the first generated selector will be identical to the extendee,
but it may need to be modified when dealing with the pseudo-selector `:not()`.
For example, `extend(:not(.foo), .foo, .bar)` should produce
`:not(.foo):not(.bar)`.

#### The Second Law

The second law of extend says that the specificity of a new selector to match a
given extender must be greater than or equal to the specificity of that
extender. For example, `extend(a, a, a.foo)` should produce `a, a.foo` even
though (again) `a.foo` matches a subset of elements matched by `a`.

This still leaves room for optimizations. For example,
`extend(.bar a, a, a.foo)` can just produce `.bar a` (omitting `.bar a.foo`).
This is allowed because `.bar a` matches a superset of the elements matched by
`.bar a.foo`, *and* the specificity of `.bar a` is equal to that of the extender
`a.foo`.
