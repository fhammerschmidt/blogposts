---
title: Reason(React) Best Practices - Part 2
published: true
description: This article provides some best practices for writing ReasonML (BuckleScript) and ReasonReact, specifically compiler flags and the Belt standard library.
tags: reason, reasonml, reasonreact, bucklescript
---

Welcome back to the second part of my blog mini series (the first one is [here]({% link fhammerschmidt/reason-react-best-practices-2cb7 %})) which kinda derived from my talk at [@reasonvienna](https://twitter.com/reasonvienna). As promised, the focus of this part is on compiler configuration and the Belt standard library.

## Compiler configuration

Well, it turns out that I am not the first person writing about BuckleScript's compiler flags here on [dev.to](https://dev.to). [Yawar Amin](https://dev.to/yawaramin) already did a great job with his post ["OCaml/ReasonML best practice: warnings and errors"](https://dev.to/yawaramin/ocaml-reasonml-best-practice-warnings-and-errors-4mkm). Still, I have a different view (and only focus on BuckleScript with ReasonML and not native, nor OCaml).

Furthermore, which warnings you really want to add or remove from the defaults is still highly subjective and may depend on you and your team's specific needs, toolchain and so on.

### Warnings ⚠️

BuckleScript's compiler warnings can be modified by prepending `+` or `-` to a warning number you want to activate or deactivate. No matter whether you just modify one or many warnings, it is always a single string you assign to the `"number"` property in the `"warnings"` object of your project's `bsconfig.json`. Here is an example with one activated and one deactivated warning:

```json
  // bsconfig.json
  "warnings": {
    "number": "+102-105"
  },
```

There is also a special `A` modifier, which means ALL of them - so if you for example put a `+A` modifier before everything else, it only makes sense to deactivate other warnings, because really all of them are now activated. We at [cca.io](https://cca.io) don't use that, though. We just use the two you see above. There are even more letter-warnings, but most of them are just aliases for existing warning numbers or groups of warnings. Finally, there is an `@` modifier, which does the same as `+` and also makes the compiler treat that specific warning as an error. If you want to list the default configuration, you can print that on the terminal via `bsc -help` and look under `-w` (or use `bsc -help | grep "+a"` if you are impatient). It yielded me the following on version 5.2.0:

```bash
$ bsc -help | grep "+a"
  default setting is "+a-4-6-7-9-27-29-32..39-41..42-44-45-48-50-102"
```

At the time of writing this post, BuckleScript comes with a whopping 62 availabe compiler warning types which have been inherited from OCaml, whereas they add another five, starting with the number `101`. Here are those five BuckleScript-specific warning types (I found them in one place in the [sources](https://github.com/BuckleScript/bucklescript/blob/5.2.0/lib/4.06.1/bspp.ml#L3393) only, and tried to derive descriptions for them based on my usage experience).

|  ⚠️ | Type                      | Default | Description                                                                                                                                                                                                                             |
| --: | ------------------------- | :-----: | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 101 | Bs_unused_attribute       |    +    | When a BS decorator attribute can not be applied to code underneath ([Try](https://reasonml.github.io/en/try?rrjsx=true&reason=NoAQRgzgdA9mBWBdAUAGwKYBcAEFsF5sAiACQFEBNIgbiA)).                                                        |
| 102 | Bs_polymorphic_comparison |    -    | Disallow polymorphic comparisons (i.e. using `==` on complex data structures).                                                                                                                                                          |
| 103 | Bs_ffi_warning            |    +    | When a foreign function interface (FFI annotation == BS decorator) is missing. ([Try](https://reasonml.github.io/en/try?rrjsx=true&reason=NoAQRgzgdAbghgGwLoCgCmAPALmgTgO0QAIAzALiIAoA-OC4IgHyIAM5KBLfLASiR6IBeAHxEIWXFwDmQogCI5AbiA)). |
| 104 | Bs_derive_warning         |    +    | When using `[@bs.deriving]` on an unsupported type ([Try](https://reasonml.github.io/en/try?rrjsx=true&reason=NoAQRgzgdAJgpgJwJYDckDsDmACAVhAYQHt0VEAXRAXQChyBPABzmwEMBuGoA)).                                                          |
| 105 | Bs_fragile_external       |    +    | When using the empty `""` shorthand in external declarations to infer the JS function name from the Reason one.                                                                                                                         |

As you can see, there is only one warning type deactivated by default, which is 102: _Disallow polymorphic comparisons_. We activated this one because when we just started writing apps in ReasonML, we used it heavily to compare deeply nested records. At some point, we stumbled over bugs where things were not equal in our minds, but the language still evaluated them so. Thus, we activated 102 and found some more possible pitfalls in our codebase. If you use it, you need to be prepared to write your own comparison functions for your data structures, or at least use the compare functions provided by the Pervasives module, or Belt's `List.cmp`, et cetera. Luckily, we are by far [not the only ones with that mindset](https://discuss.ocaml.org/t/removing-polymorphic-compare-from-core/2994). It is also recommended to turn it on by the [creator of BuckleScript himself](https://reasonml.chat/t/polymorphic-compare-in-reasonml-bucklescript/1294/7?u=fham), at least when you consider yourself experienced enough.

We also deactivated 105 for a current project, though that was more due to laziness to not have to change hundreds of lines of JS bindings. There were just too many warnings to cope with, after the [BuckleScript upgrade to version 5.0.5](https://bucklescript.github.io/blog/#a-new-warning-number-105).

### Errors ❌

All warnings can also be treated as errors. This may disturb the developer experience, but in a continuous integration context it can be pretty useful. The programmer only needs to ensure to push code without warnings, so that the app will be built. If you feel like a 10x engineer, you can also activate it in your local coding setup. 😉

Just add the corresponding string of warnings you want to let the compiler fail on to the `"error"` property, like so:

```json
  // bsconfig.json
  "warnings": {
    "error": "+8+11+12+26+27+31+32+33+34+35+39+44+45+102",
  },
```

The string above also happens to be the one we actually use in CI.

### Flags 🏁

There is another very useful property in the `bsconfig.json`, the `"bsc-flags"`, those are the parameters with which the `bsc` compiler is invoked by default. Most of them have pretty sane defaults, but it is still nice to know that there are lots of knobs to finetune the behaviour of the compiler. As it is the case with warnings, there are also flags derived from OCaml as well as BuckleScript-specific ones, which are prefixed with `-bs`.
`bsc --help` lists all the possible options (a load of them!), so check them out if you are interested.

Personally, I only scratched the surface of all those possibilities, but for instance you can deactivate the `bs` header in the compiled `.bs.js` files via `-bs-no-version-header`.
It looks like that at the top of every generated file, by default:

```js
// Generated by BUCKLESCRIPT VERSION 5.0.4, PLEASE EDIT WITH CARE
```

We omit them to be able to compare output of different compiler versions (useful for beta-testing of new BS releases), and other than that, we do not feel that it's very necessary for an app you want to ship. It may be a different story for libraries which get consumed by JS.

Another useful flag is `-open` which requires a module name, e.g. `-open Belt`. This opens the given module in every Reason (or OCaml) file you are editing, so there is no need to put an

```reasonml
open Belt;
```

at the top of most files. This is especially useful, if you have a common module you want to share over the whole codebase, like some utility functions. In the case of `Belt`, you also cannot make the mistake anymore to use a built-in `List` or `Array` method, instead of the more idiomatic, pipe-first and camelCase `Belt` counterparts. This is only recommended if you are sure that you will never use the (OCaml-ish) built-ins. Personally, I don't do that yet, but I also don't really use the built-ins and could easily do the jump any time.

Speaking of `Belt`, let's head to the next chapter...

## The Belt stdlib

When I started with Reason, I was overwhelmed by the possibilities of both the OCaml stdlib and the Belt one (Belt was also less extensive at that time). It also did not help, that you are often forced to read interface files as documentation in the OCaml world. It may be efficient, because types need to be correct which is not always the case for documentation, but if you want to lure aspiring JS devs on your type-safe boat, nothing is nicer than a pretty, tidied documentation, possibly with examples which let you dive in directly.

Luckily, the [Reason Association started a big project](https://www.reason-association.org/projects/better-learning-materials-and-tools) where, among other things, the Belt documentation will be overhauled and vastly improved, extended with examples, et cetera. If you are interested, you can have a look at the current status over at the future home of ReasonML, [reasonml.org](https://reasonml.org) or even help out and submit a PR (there are some [good first issues](https://github.com/reason-association/reasonml.org/issues)).

Also, here on dev.to, [John Jackson](https://dev.to/johnridesabike) already wrote [a pretty good series about Belt's containers](https://dev.to/johnridesabike/introduction-to-bucklescript-s-belt-containers-map-hashmap-set-3b5l), which can also help at least until the docs are finished.

### The SortArray module

There is one tidbit I want to share here, and that is about how to sort Arrays with Belt. Becaues you may wonder where the sort method is (spoiler: it's not in the `Belt.Array` module). To keep it short: it is right in the root of the `Belt` module, namely `Belt.SortArray`. If you have an array of records, for instance users you want to sort by their last name, the idiomatic way is to provide a compare function for the `User` record type. The simplest compare function is a `-` for `int`, but as you learned above, a stricter approach is preferable:

```reasonml
module User = {
  type t = {
    firstName: string,
    lastName: string,
    id: string,
  };

  let compare = (a: t, b: t) => String.compare(a.lastName, b.lastName);
};
```

In the above case, we just use a simpler part of the record (the `lastName`, which is a `string`), to compare the whole user record, for instance to view it in a table sorted by last name. The `String.compare` function is out of the `Pervasives`, so it is available by default. Compare functions have to have the same signature as `Pervasives.compare` and return `-1` (or any negative integer) when `a < b`, `1` (or any positive integer greater than 0) when `a > b`, and `0` when they are equal.

This is also the kind of function which can be consumed by `SortArray`, like so:

```reasonml
let sortUsers = (users: array(User.t)) => users->SortArray.stableSortBy(User.compare);
```

In this case, we use stableSortBy, which is one of the most common methods of the module, but there are many more, like `intersect`, `diff`, `union` or `binarySearch`.
Feel free to try this on an array of users in the [Reason Try playground](https://reasonml.github.io/en/try?rrjsx=true&reason=PYBwpgdgBAQmA2AXA3AKFQW2AEwK7zCgFUBnMAJygF4oBvVKKRAT3CerocagDMBLciUQA5AIYYwALihDyfCAHMANF0bxRQsROmz5y1VD7YdiOYpWMAvmi4FEUAMbAMIUeUI0AFKOmIlUACNfAEpqAD4oAGVTPQA6Jxc3MG9Y9U1xMH8A1I0RDOC0a3Q7GWByRFIKEg5PXDJBaTdyUWZPSvJYxGDQqgi6qoBaMMiyxABBcmbmWKFRAIIR8phW9vjnV3cC4rB7fsEOAG0AHy5aVf5BPO0oACIAUXh4PmBEG-80q6lbsfhsKuAIG9DMZbgBmACcABYABw3SwWOgXdLXG4AFWY7keQI+Wi+NwA6ggng4ANZAozSG6QgBM4IADHCEbQkZ9KWNFAhRNjcrjKQBZYAkEjkkE3ACM1LpAHZGagjgBdGx7EhDEijdoqsIAKRIqWACmQQA).

And that's all for this issue of my Reason(React) Best Practices blog. Next time I will shed some light on how to write zero-cost bindings to JS modules with BuckleScript.
