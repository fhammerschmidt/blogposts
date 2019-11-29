---
title: Reason(React) Best Practices - Part 3
published: false
description: This article provides some best practices for writing ReasonML (BuckleScript) and ReasonReact, specifically about writing zero-cost bindings.
tags: reason, reasonml, reasonreact, bucklescript
---

Welcome back to the third part of my blog mini series ([Part 1](https://dev.to/fhammerschmidt/reason-react-best-practices-2cb7), [Part 2](https://dev.to/fhammerschmidt/reason-react-best-practices-part-2-2opc)) which derived from my talk at [@reasonvienna](https://twitter.com/reasonvienna). This part is all about writing zero-cost bindings to existing JavaScript functions and React components.

## Background

Since ReasonReact included Hooks support starting with version `0.7.0`, writing React in Reason has become a breeze. The library now became a very thin layer between React and BuckleScript and as a result, we did not need to write tedious boilerplate code anymore (mostly to map to the behaviour of class-based React components). But before the old way gets completely lost in history, here is a small reminder how we wrote React in ReasonML just some months ago:

```reasonml
let component = ReasonReact.statelessComponent("Greeting");

let make = (~name, _children) => {
  ...component,
  render: self =>
    <h1> {ReasonReact.string("Hello " ++ name)} </h1>,
};
```

vs.

```reasonml
/* Greeting.re */
[@react.component]
let make = (~name) => <h1> {React.string("Hello " ++ name)} </h1>
```

To be fair, some magic happens in the `react.component` annotation, but it was just more work to create a stateless component in which we needed to spread our component definition into. Not to speak of comparing stateful components, the differences are humungous. For compatibility reasons, we still have the old module (`ReasonReact`), but the [new one](https://github.com/reasonml/reason-react/blob/v0.7.0/src/React.re) (just `React`) is about a third of the size of the [old one](https://github.com/reasonml/reason-react/blob/v0.7.0/src/ReasonReact.re).

## Penalty-free bindings

Some of the most important differences though, are the implications on performance. Whereas the old ReasonReact comes with its own React Mini to be compatible with class-based components, the new one is meant to be a very thin layer between React and ReasonML, (mostly) without any additional performance cost. To be fair, this is only true if you use the Hooks API. If you rather write class components, then you still have to use the old approach (but then you probably would not use the FP language Reason either).

This idea also spilled over to the [React Native bindings](https://reasonml-community.github.io/reason-react-native/en/docs/), which are mostly done at this point in time (only some of the more obscure or deprecated ones are missing).

I think this attitude is a good one to have, because we do not want to lose performance-wise against plain JS - it would make newcomers more hesitant to dive into the ReasonML ecosystem. Even better: by using the right annotations, you get the most out of your bindings and win some developer experience too. For instance if you take `[@bs.string]` which converts plain string enums into polymorphic variants (the ones which start with a `` `backtick ``) and in consequence make the compiler restrict your code to using only those variants.

In the following section, I will examine one of the well-written React-Native bindings, by using example apis or components directly from the [sources](https://github.com/reason-react-native/react-native/tree/master/src).

### React-Native Alert

The Alert API consists of two callable methods, `alert` and `prompt`. While `prompt` actually does only trigger on iOS devices (and is not even mentioned in the official React Native docs, as of yet), we still want to bind that too for educational purposes. If you look at the [JS sources](https://github.com/facebook/react-native/blob/0a66ded7d006558213681f8efbdb762cbadfcc08/Libraries/Alert/Alert.js#L19) of `Alert`, you will notice the Flow types which make it pretty straightforward to translate everything to Reason code.

> **Note 1**: As the following section contains both JS and ReasonML code, a small comment header will tell you what language I am talking about.

> **Note 2**: The Reason source code may be formatted differently than what `refmt` would put out. I adapted it for readability and conciseness.

The first type is `AlertType`. It is a string enum consisting of

```js
/* JS */
  | 'default'
  | 'plain-text'
  | 'secure-text'
  | 'login-password';
```

this translates to

```reasonml
/* Reason */
type alertType: [@bs.string] [
              | `default
              | `plainText
              | `secureText
              | `loginPassword
            ];
```

Note that you cannot use a hyphen in polymorphic variants, so there is another step necessary:

```reasonml
/* Reason */
type alertType = [@bs.string] [
              | `default
              | [@bs.as "plain-text"] `plainText
              | [@bs.as "secure-text"] `secureText
              | [@bs.as "login-password"] `loginPassword
            ];
```

[@bs.as] takes care of that. Generally, when there are hyphens, special characters, or numbers at the start, this approach is needed.

There is also the `AlertButtonStyle`, which can be translated in the same way:

```js
/* JS */
export type AlertButtonStyle = "default" | "cancel" | "destructive";
```

translates to

```reasonml
/* Reason */
type alertButtonStyle = [@bs.string] [ | `default | `cancel | `destructive],
```

Then there is `Buttons`, an array of `Button`. I think it is better to define a button type and then define a buttons array type instead of mixing them all into one type definition, so let's pretend that the JS types are also written like that:

```js
/* JS */
export type Button = {
  text?: string,
  onPress?: ?Function,
  style?: AlertButtonStyle
};
```

becomes

```reasonml
/* Reason */
type button; // create abstract button type

[@bs.obj]
external button: // define a translator function for the button type
  (
    ~text: string=?,
    ~onPress: unit => unit=?,
    ~style: alertButtonStyle=?,
    unit
  ) =>
  button = // returns a button
  ""; // dummy placeholder
```

This one is more complex, so we use `[@bs.obj]` here. It lets us define a function from which the compiler derives a JS object. This is useful if you have many optional parameters, as it is the case with the button above. The `=?` after the corresponding type denotes that the function is not required.

Then there is the `options` type:

```js
/* JS */
type Options = {
  cancelable?: ?boolean,
  onDismiss?: ?() => void
};
```

which works the same as the button above:

```reasonml
/* Reason */
type options;

[@bs.obj]
external options:
  (
    ~cancelable: bool=?,
    ~onDismiss: unit => unit=?,
    unit
  ) => options = "";
```

Note that the type information of `onDismiss` is much clearer to translate - an empty function body yielding `void` which becomes `unit => unit` in Reason land. For translating `?Function` one needs to look up the flow docs which say `Function` is basically `any`, so we only know what it really returns by using it and logging things out or digging through the source code. But in this case, we know it is still `unit => unit`.

The final piece is the signature of the method itself. As we speak of a JS class component, the `alert` method's signature can be found under `static alert(...)`:

```js
/* JS */
static alert(
  title: ?string,
  message?: ?string,
  buttons?: Buttons,
  options?: Options,
): void
```

which translates to

```reasonml
/* Reason */
[@bs.scope "Alert"] [@bs.module "react-native"]
external alert:
  (
    ~title: string,
    ~message: string=?,
    ~buttons: array(button)=?,
    ~options: options=?,
    unit
  ) => unit = "";
```

The most important part here is the first line - we need `[@bs.module]` to tell the compiler that we want to call a method from an external module, in this case from `"react-native"`. Also we want to look it up under the `Alert` module of React Native. Therefore we utilize `[@bs.scope]` with `"Alert"`. Remember, this is the equivalent of doing

```js
import { Alert } from "react-native";
```

in JavaScript and I think that it is mapped pretty well that way.

The signature of the `prompt` method is also typed:

```js
  static prompt(
    title: ?string,
    message?: ?string,
    callbackOrButtons?: ?(((text: string) => void) | Buttons),
    type?: ?AlertType = 'plain-text',
    defaultValue?: string,
    keyboardType?: string,
  ): void
```

This also looks doable:

- `title` and
- `message` work the same way as in the `alert` method,
- `defaultValue` and
- `keyboardTypes` are just strings,
- `type` is already typed above, known as `alertType`.

So easy going, right? But what abomination of a property is `callbackOrButtons`? This is a JavaScript'ism which I would even call an antipattern in ReasonML. In the Reason (and FP) world, you would rather statically check your types and not inspect whether your prop is an array and do something differently than what you would have done with a function, all at runtime. This is such a thing which can only be done in a dynamically typed language. \*_hissing snake noises_\*

But be assured, that even for such cases, BuckleScript provides a wonderful remedy: [@bs.unwrap]. It utilizes polymorphic variants which all get compiled away, so we don't lose our precious performance. We just have to create two of them, one for `` `callback `` and one for `` `buttons ``. The button type has been defined already above and the callback just translates easily from Flow again, `string => void` becomes `string => unit` in Reason land.

```reasonml
    ~callbackOrButtons: [@bs.unwrap] [
                          | `callback(string => unit)
                          | `buttons(array(button))
                        ]=?,
```

So we end up with this:

```reasonml
[@bs.scope "Alert"] [@bs.module "react-native"]
external prompt:
  (
    ~title: string,
    ~message: string=?,
    ~callbackOrButtons: [@bs.unwrap] [
                          | `callback(string => unit)
                          | `buttons(array(button))
                        ]=?,
    ~type_: [@bs.string] [ /* alertType */
              | `default
              | [@bs.as "plain-text"] `plainText
              | [@bs.as "secure-text"] `secureText
              | [@bs.as "login-password"] `loginPassword
            ]=?,
    ~defaultValue: string=?,
    ~keyboardType: string=?,
    unit
  ) => unit = "prompt";
```

Of course writing bindings is only half the fun, so here's is an example of how you would use the `Alert.alert` binding:

```reasonml
Alert.alert(
  ~title="Warning!",
  ~message="Do you want to delete the entry?",
  ~buttons=[|
    Alert.button(~text="Delete", ~onPress, ~style=`destructive, ()),
    Alert.button(~text="Cancel", ~style=`cancel, ()),
  |],
  ~options=
    Alert.options(
      ~cancelable=false,
      ~onDismiss=_ => Js.log("Deletion aborted."),
      (),
    ),
  (),
);
```

And here's is an example of how you would use the `Alert.prompt` binding (again, only on iOS):

```reasonml
Alert.prompt(
  ~title="Enter Password",
  ~message="Please enter your password.",
  ~callbackOrButtons=`callback(text => Js.log(text)),
  ~type_=`secureText,
  (),
);
```

Don't forget to call all the defined `external`s with their Module name (here `Alert`) before or `open` it in the scope.

Speaking of externals, a good rule of thumb to know whether you created some runtime overhead with your bindings, is the following:

> Have a look if there are any `let`s in your code, rather than `external`s.

To be sure look into your created `.bs.js` bindings file. When you only see

```js
/* This output is empty. Its source's type definitions, externals and/or unused code got optimized away. */
```

you can pat yourself on the shoulder, because you successfully created a zero-cost binding!

That's all for my mini series about Best Practices in Reason & ReasonReact, at least for now. Any upcoming posts will not be part of this series anymore, but they will almost certainly contain ReasonML stuff.

So have a great time, and make great (type-safe) things!
