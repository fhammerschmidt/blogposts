---
title: Reason(React) Best Practices
published: true
description: This article provides some best practices for writing ReasonML (BuckleScript) and ReasonReact, specifically the identity trick, pipes and labelled arguments.
tags: reason, reasonml, reasonreact, bucklescript
---

After using Reason with React for nearly 2 years, I decided to hold a talk about best practices with Reason and ReasonReact at the ReasonML meetup in Vienna ([@reasonvienna](https://twitter.com/reasonvienna)). Given that it was my first tech talk in a tech meetup, it went pretty well. I published the slides that same day on Twitter, but slides unfortunately do not tell the full story. So I decided to give it a go and put it here in a more descriptive form. I also split it up a bit, as the talk took about one hour and I prefer to keep blog posts short and to the point.

But enough waffling, let's get into it:

## The identity trick

### Zero-cost React.strings!

With Reason being a 100 % type-safe language, there are some peculiarities about writing JSX in Reason which make life for beginners hard, especially when they come from JS and are used to write strings directly into a `<div />` or any other React element which allows children. In Reason, the `React.string` method is needed everywhere where you want to render text in your app, so it makes sense to create a binding for it to a one-letter function.

You could do that by writing

```reasonml
let s = React.string;
```

but you can utilize the compiler to optimize all unnecessary parts away, by using the `external` keyword:

```reasonml
/* Shorter replacement for React.string */
external s: string => React.element = "%identity";
```

To compare the methods, check out this example in the ReasonML playground: [ReasonML Try - 1](https://reasonml.github.io/en/try?rrjsx=true&reason=LYewJgrgNgpgBAMQJYCcDOAXOBeOBvAKDjlizRzgCUYBDAYwwDpMUkA7AcwG4Ci4BtAAIpaDRnRDAADiDYw2GALp9ScYDQDW8XAAoAlDgB8+PsWIAeMEgBuxvGh0AiZOgyO9AXzjmA9Fdt8HjxBvKCQsHAAyjASbGAUhMQwAB4YMChsNFBwaABcORisnEZUokwwsMDyWLiOAKRIYNVIGACejjzEfEIi9EwS0rLVysSq6loU+iWJZhb+dg6O0bFg7l6+84HBXEA).

As you can see, there is still a function `s` being created in the first example, whereas the second one is nowhere to be found. It's probably a very very small runtime improvement, but it gives a good idea of how the BuckleScript compiler works. And if you look at the sources, it is actually the same thing as the provided `React.string` from [ReasonReact](https://github.com/reasonml/reason-react/blob/v0.7.0/src/React.re#L5).

In either way, you probably should put the helper function in a utilities module which you can `open` everywhere you need those precious strings in react elements, like so:

```reasonml
open ReactUtils;

[@react.component]
let make = () => {
  <div>
    {s("Hello, dev.to!")}
  </div>
};
```

I named the module `ReactUtils`, because it is specifically meant for working with React elements.

### Keep your codebase free of magic!

Sometimes the type system is just too strict, and in a heterogenous software system there may exist types which are completely the same thing, only with different names. With the `identity` trick you can convert types when they would be actually the same, for instance `Js.Dict.t(string)` and `Js.Json.t`:

```reasonml
type apples = Js.Dict.t(string);
type oranges = Js.Json.t;

let (apples: apples) = Js.Dict.fromArray([|("taste", "sweet")|]);

let eatFruits = (fruits: Js.Json.t) => Js.log2("Eating fruits", fruits);

eatFruits(apples);
```

As you can see, we have apples and oranges here. Both are fruits, as commonly known, but only oranges are actually also defined as `Js.Json.t`. Apples however are defined as `Js.Dict.t(string)` which makes the compiler throw an error (see [ReasonML Try - 2](https://reasonml.github.io/en/try?rrjsx=true&reason=C4TwDgpgBAhmYBsIGcoF4oClkDoAiAlgMbA7AAUywATgQHYDmAlANwBQokUA9tTIynRZc2bnTLs2SYFHJxEKAFyx4SZEyHZ8xUgDNq3ALYBBanxDkA2gB9yAImAwqEOwBood5AHcIEYHaZrAF1WNik-KAgYYAAxagBXAmBUDHJ9ROTlLVFxYA00AD5hHARuBgAmewBRaPoGKHSk5DcGhKbQtijYtuS5VRRQoA)).

The easiest way to make this code compile is to use `Obj.magic`. It basically switches the type checker off and lets the compiler do his job.

```reasonml
eatFruits(apples->Obj.magic);
```

([ReasonML Try - 3](https://reasonml.github.io/en/try?rrjsx=true&reason=C4TwDgpgBAhmYBsIGcoF4oClkDoAiAlgMbA7AAUywATgQHYDmAlANwBQokUA9tTIynRZc2bnTLs2SYFHJxEKAFyx4SZEyHZ8xUgDNq3ALYBBanxDkA2gB9yAImAwqEOwBood5AHcIEYHaZrAF1WNik-KAgYYAAxagBXAmBUDHJ9ROTlLVFxYA00AD5hHARuBgAmewBRaPoGKHSk5DcGhKbQtijYtuS5VRQAWgKAeQAjACscQxgGYlCgA))

`Obj.magic` is actually also implemented by using the identity trick:

```reasonml
external magic : 'a => 'b = "%identity";
```

(see the [source code](https://github.com/BuckleScript/ocaml/blob/698e60f3cd2f442f2013e79860ce2f7b0a9624cb/stdlib/obj.ml#L20)).

But it is both more idiomatic (and less risky) to write a conversion function for the two specific types:

```reasonml
type fruits = Js.Json.t;
external applesToFruits: apples => fruits = "%identity";

eatFruits(apples->applesToFruits);
```

This lets your code compile and still ensures that there is only one distinct type consumed by the function and not _everything_. It still may make sense to use `Obj.magic` sometimes, especially when you just want to create a quick prototype (see [ReasonML Try - 4](https://reasonml.github.io/en/try?rrjsx=true&reason=C4TwDgpgBAhmYBsIGcoF4oClkDoAiAlgMbA7AAUywATgQHYDmAlANwBQokUA9tTIynRZc2bnTLs2SYFHJxEKAFyx4SZEyHZ8xUgDNq3ALYBBanxDkA2gB9yAImAwqEOwBood5AHcIEYHaZrAF1WNik-KAgYYAAxagBXAmBUDHJ9ROTlLVFxYA00AD5hHARuBgAmewBRaPoGKHSk5DcGhKbQjnBoRuTNEWQxCTYIAA9gCGo6GAQVBWQAFW44jORleTV0Ip6UjwBSAgATCDpgJJA7SSjYtuS5VRQAWgL1lEXl9vYgA)).

## Pipes

### Use the Pipes like Mario!

As many (functional) languages do, also Reason provides us with a special pipe operator (`->`), which essentially flips the code inside out, e.g. `eatFruits(apple)` becomes `apple->eatFruits`. At first it was hard for me to read and comprehend longer pipe chains, but I got used to it after some days of using them. Now they are one of the most indispensable features to me.

- Keeps you from needing to find a name for in-between variables.
- Keeps code tidy.
- Especially useful with `Belt` (BuckleScript's standard library) and its `Option` module which you will encounter very often as we have no `null` or `undefined` here in Reason land.
- When using pipe-first (`->`) with, say `Array.map`it makes the code look pretty similar to Js, e.g.: `[|1, 2, 3|]->Array.map(x => x * 2)` which would be `[1, 2, 3].map(x => x * 2)` in plain JS.

But just compare the two examples below, first one without using pipes:

```reasonml
let vehicle = Option.flatMap(member.optionalId, id => vehicles->Map.String.get(id));
let vehicleName = Option.mapWithDefault(vehicle, "–", vehicle => vehicle.name)}
s(vehicleName);
```

vs.

```reasonml
member.optionalId
  ->Option.flatMap(id => vehicles->Map.String.get(id))
  ->Option.mapWithDefault("–", vehicle => vehicle.name)
  ->s
```

### Pipe parameter position paranoia!

I know there is some war going on between the last-pipers (mostly native Reason and OCaml developers) and the first-pipers (BuckleScript developers), but in BuckleScript, because you have JS as the target language, pipe-first is the way to go. The type inference also works better that way.

If you really want to pipe into a function which is not idiomatic to BS (i.e. the positional parameter is not the first one), you can use the `_` character to substitute where the piped parameter should go:

```reasonml
"ReasonReact"->Js.String.includes("Reason", _);
```

but rather use the pipe-first version of a function preferably, as long as it exists:

```reasonml
"ReasonReact"->Js.String2.includes("Reason");
```

as mentioned earlier.

## Labels to the rescue!

When you have a function like `String.includes` where multiple parameters have the same type, it would be much better to label them directly, otherwise you won't know which of the parameters is the source string, and which one is the search string:
![string.includes](https://thepracticaldev.s3.amazonaws.com/i/7v33vi0zr2xmoz9j5ira.png)

Even worse, if you use the wrong pipe (`|>`) and are unsure which parameter it takes, you can get confused easily. And the compiler cannot save you from that case, as the types are totally correct.

Here is a `String.includes` function binding with labels to keep you from guessing which one the positional parameter is:

```reasonml
module BetterString = {
  [@bs.send] external includes : (string, ~searchString: string) => bool = "includes";
};

"ReasonReact"->BetterString.includes(~searchString="Reason");
```

([ReasonML Try - 5](https://reasonml.github.io/en/try?rrjsx=true&reason=LYewJgrgNgpgBAIRgF2TATgZWeglgOwHM4BeOAbwCg44BtAAQCMBnAOmZnzAF04YAPNOnwBDKHAIBjKBDAxmcAFxwAFMxwFCAGjgA-DiPSSAFtjxFl684QCUpAHxxGIEOLIAiKTLnN3AbkoAXwDKdwAlGBFmEHwIkUlkdwBaeyRUDDNNVi9ZeRVwyOj8dxsUgCk2KBBCPyA))

To be fair, we now need to type a bit more, but we gain some doubtlessness and do not have to check MDN to be sure. Also, when you ignore warnings or deactivate the corresponding compiler warning (it's number 6), you could still call the function without labelling the parameter. I would still advise you to not do that, though.

That's all for the first part of _Reason(React) Best Practices_, our next topic will be about BuckleScript's compiler configuration. and Belt.
