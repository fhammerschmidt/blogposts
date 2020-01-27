---
title: What's new in BuckleScript 7?
published: true
description: A tour over the new features and changes of BuckleScript 7.
tags: reason, reasonml, bucklescript
---

Another groundbreaking change for all ReasonML/BuckleScript developers has emerged this week, because BuckleScript 7 has been [officially released](https://reasonml.chat/t/ann-bs-platform-7-0-0-is-released/2056/1). The maintainer, Bob Zhang, does a great job in announcing the most important new features and changes on the [official BuckleScript blog website](https://bucklescript.github.io/blog/), but I want to cover them in a bit more detail here. Of course, the full changelog can always be viewed at the [repository](https://github.com/BuckleScript/bucklescript/blob/master/Changes.md#701), but it's a bit raw to digest and there are some great things hidden behind all those pull requests.

Allow me to also include some important changes since version 5.2.1, because many people did not even make the jump to v6 already (including me). But v7 has so many good things to offer that the relatively minor risks are outweighed by a bunch of improvements.

Before I get into it, just a heads up: We upgraded a big codebase to from 5.2.1 to 7.0.1 and had only a very minor [issue](https://github.com/facebook/reason/issues/2507). We still released it compiled via BuckleScript 5, because there was just not enough time for thorough testing, but it compiled, no tests failed, and some manual testing has also been successful.

But we also did not use any [PPXes](http://ocamllabs.io/doc/ppx.html), which may be a blocker for others. This was already the biggest hurdle for people switching to v6, since it is based on OCaml 4.06, while v5 is based on OCaml 4.02. The new BuckleScript 7 will still be based on OCaml 4.06, so let's discuss the differences between those two versions first.

# Changes in V6

## Inline Records

Inline records can now be used as [arguments to datatype constructors in sum types](https://caml.inria.fr/pub/docs/manual-ocaml/manual040.html). This is very convenient, for instance when you have a variant type which includes a record in one case, but do not want to create a separate type for the record just for this one variant case. Previously, you were forced to declare a new record type like this:

```reasonml
type point = { width: int, mutable x: float, mutable y: float};

type t =
  | Point(point)
  | Other;

let v = Point({width: 10, x: 0., y: 0.});
```

which can be achieved with fewer lines of code now:

```reasonml
type t =
  | Point({ width: int, mutable x: float, mutable y: float})
  | Other;

let v = Point({width: 10, x: 0., y: 0.});
```

Nice!

Well, to be fair the old (v5) compiler gave you an useful error when you used inline records, at least:

```
refmt: internal error, uncaught exception:
  File "/[blogposts]/bs-7-example/src/InlineRecord.re", line 2, characters 4-9: inline records are not supported before OCaml 4.03
```

## Internal Result Type

OCaml baked the `result` type with 4.04 into the language. `result` is one of the most beloved features about ML and related languages for try-catch-ridden former OOP-developers who found their new home in FP languages. I mean, you can even convert exceptions into less threatening `Result.Error`s and everything is fine. On a more serious note, it always bothered me that you need to `open Belt` to have a result type at your disposal, because it is equally fundamental to a modern FP language as the option type (in my humble opinion).

This means that you can do this:

```reasonml
let bustAMove = move =>
  switch (move) {
  | Ok(bust) => Js.log(bust)
  | Error(error) => Js.log2("Can't bust a move because of: ", error)
  };
```

without opening `Belt` or any other library which provides a `result` type.

## Local opening of modules in a pattern

Local opening of modules can be very useful to prevent name clashes. It's the syntax where you put a `.` and `()` after a module name to open the module only for what's inside the parentheses, e.g.:

```reasonml
Sausages.(pork->meatGrinder->fillIntoSkin)
```

You can also use that in a pattern now!

```reasonml
module CallMom = {
  type t = {
    smalltalk: string,
    story: option(string),
  };
};

let callMom = call =>
  switch (call) {
  | CallMom.{story, smalltalk} => Js.log2(story, smalltalk)
  | CallMom.{smalltalk} => Js.log2("Only gossip today", smalltalk)
  };
```

If you tried that with the old BuckleScript, it would yield an error.

That's all for the changes introduced with BuckleScript 6, let's move on to even more interesting grounds.

# Changes in V7

## Refmt Update / Re-Error

There has been a lot of work put into a better code parser to get better error messages which was a common pain point for many newcomers. The so-called Re-Error coming with the current ReasonML parser & formatter (refmt) makes the developer experience a bit better.

Consider the following very simple line of code:

```reasonml
let x = {
  foo;
```

When you compile that, with older refmt versions you only get a

```
File "/[redacted]/bs-7-example/src/SampleError.re", line 2, characters 2-6:
Error: SyntaxError in block
```

whereas now the message is much more helpful and descriptive:

```
File "/[redacted]/bs-7-example/src/SampleError.re", line 2, characters 6-6:
Error: Unclosed "{" (opened line 1, column 8)
```

Even for more complex errors, it is a great relief that it can still help you out. Imagine having a big tree of JSX functions and a missing closing `>` somewhere in the tree:

```reasonml
type state = {count: int};

type action =
  | Increment
  | Decrement;

let initialState = {count: 0};

let reducer = (state, action) =>
  switch (action) {
  | Increment => {count: state.count + 1}
  | Decrement => {count: state.count - 1}
  };

[@react.component]
let make = () => {
  let (state, dispatch) = React.useReducer(reducer, initialState);

  <main>
    {React.string("Simple counter with reducer")}
    <div>
      <button onClick={_ => dispatch(Decrement)}>
        {React.string("Decrement")}
      </button>
      <span> {state.count |> string_of_int |> React.string} </span // <-- Here we miss a closing '>'
      <button onClick={_ => dispatch(Increment)}>
        {React.string("Increment")}
      </button>
    </div>
  </main>;
};
```

(The example is from [reason-minimal-template](https://github.com/MargaretKrutikova/reason-minimal-template), thanks [Margarita](https://dev.to/margaretkrutikova)!)

BuckleScript 5 cannot deal with this error:

```
File "/[redacted]/bs-7-example/src/App.re", line 17, characters 2-66:
Error: SyntaxError in block
```

whereas BuckleScript 7 is much better:

```
File "/[redacted]/bs-7-example/src/App.re", line 25, characters 60-62:
Error: syntax error, consider adding a `;' before
```

At least the correct line has been found in version 7. The suggestion to add an `;` is valid for normal code, but not the syntactic sugar we commonly know as JSX. Still, no more searching for the correct line, you're directly pointed to it and probably can then easily figure out for yourself what's wrong.

I wanted to make the example simpler initially (i.e. with only setState which means no separately declared types, intitialState and reducer), but then also the old BuckleScript 5.2.1 found the error. The difference is still night and day though, because in my experience, most components turn out more complex than this simple example.

The work on Reerror is not finalized yet. The roadmap can be found here: [Reerror-Plan](https://github.com/facebook/reason/blob/master/PLAN), but it's nice to know that it will continue to improve over time.

## Records as JS Objects

Probably the most impactful change coming with BuckleScript 7 is that now records are not compiled to (nested) arrays anymore, but JavaScript objects. This is not only very nice for debugging, it also makes many of the interop annotations obsolete, or at least makes it more straightforward to write bindings.

### Easier Debugging

For instance, previously the following record:

```reasonml
type person = {
  firstName: string,
  lastName: string,
  phone: string
};

let jordan: person = { firstName: "Jordan", lastName: "Walke", phone: "73276665" };

Js.log(jordan);
```

produced an array:

```
[ 'Jordan', 'Walke', '73276665' ]
```

which may be more performant, but for more complex records (nested records) it was really hard to find the field you were looking for. Now it produces the following log output:

```
{ firstName: 'Jordan',
  lastName: 'Walke',
  phone: '73276665' }
```

which is already a great help for debugging, but it gives us much more power.

### Easier Interop

Let's explore how we can get some cleaner syntax by updating some bindings. As an example, let's take NodeJS' `pathObject` which is the output of the `parse` function, as well as the input for the `format` function:

```reasonml
type pathObject;
[@bs.obj]
external pathObject:
  (
    ~dir: string,
    ~root: string,
    ~base: string,
    ~name: string,
    ~ext: string,
  ) =>
  pathObject =
  "";

[@bs.module "path"] external parse: string => pathObject = "parse";
[@bs.module "path"] external format: pathObject => string = "format";
```

Now if you wanted to format a `pathObject` into a `string`, with Bucklescript <= 7 it worked the following way:

```reasonml
let path = format(
  pathObject(
    ~dir="/[redacted]/bs-7-example/src",
    ~root="/",
    ~base="NodePath.re",
    ~name="NodePath",
    ~ext=".re",
  ),
);
```

Let's look at how to bind that in BuckleScript 7:

```reasonml
  type pathObject = {
    dir: string,
    root: string,
    base: string,
    name: string,
    ext: string,
  };
```

The methods themselves do not change their type signature, as we use the same name for the type. Calling `parse` is a bit more concise and readable now:

```reasonml
let path = format({
  dir: "/[redacted]/bs-7-example/src",
  root: "/",
  base: "NodePath.re",
  name: "NodePath",
  ext: ".re",
});
```

It really looks like JavaScript now!

Bear in mind that for functions with lots of optional parameters it's still easier to use `[@bs.obj]`, as you would otherwise need to write conversion functions for the nullable types which come at a runtime cost. Records in ReasonML always need to have all their fields set (even if it is `None` or `Js.null`).

Another benefit of records-as-objects is the ability to pattern-match on them. So now we can match on a NodeJS `pathObject` without the need of converting those objects to records:

```reasonml
let getExtensionType = path => {
  switch (path) {
  | {ext: ".re"} => Reason
  | {ext: ".ml"} => OCaml
  | {ext: ".js"} => JavaScript
  | {ext: ".ts"} => TypeScript
  | _ => Unknown
  };
};
```

Of course in this case you could also just match on `path.ext` directly. But it may be the case that you want to combine the extension with a certain name or something. I did not want the example to be too convoluted though, I hope it still supports my intention well.

Also mentioned on the [BuckleScript blog](https://bucklescript.github.io/blog/2019/12/27/whats-new-in-7-cont): you can now use the `[@bs.as]` annotation in records too, so whenever you have a `-` or any other otherwise unrepresentable character, you could always use that as an escape hatch. It works the same way as usual:

```reasonml
type httpHeaders = {
  authorization: string,
  [@bs.as "content-type"] contentType: string,
  [@bs.as "last-modified"] lastModified: string,
  [@bs.as "if-modified-since"] ifModifiedSince: string,
};

let headers = {
  authorization: "Basic thisCouldBeYourCryptoKey",
  contentType: "application/javascript; charset=utf-8",
  lastModified: "Fri, 06 Dec 2019 07:39:45 GMT",
};
```

## Are we still fast?

One point where BuckleScript never failed to deliver is build time, which has always been very fast. Incremental builds are nearly instant and also full project compilations are no opportunity to do something else in the meantime. However, this time I have to say, that build times actually take a bit longer now, but it is a small fee for a big gain.

One of our current projects is about 27k lines of Reason code, according to the following command:

```
find ./src -name '*.re' -o -name '*.ml' | xargs wc -l
```

the full, cleaned build took around **2250** milliseconds with BuckleScript 5.2.1, whereas with BuckleScript 7.0.1, it's about **2500ms**. I do have faith though, that this number will fall again with the next minor releases. The build times were tested on a MacBook Pro (15-inch, 2018), 2,6 GHz Intel Core i7, 32 GB 2400 MHz DDR4.

That's everything I have to say for today. Maybe some new patterns will emerge from this useful upgrade. I am already exited to use BuckleScript 7 in the next project, what about you? Whenever you are ready to upgrade, consider reading the official [upgrade guide to v7](https://bucklescript.github.io/docs/en/upgrade-to-v7) first or contact other Reasonauts and BuckleScriptians in the [ReasonML discord](https://discord.gg/reasonml) for help.

Happy coding!
