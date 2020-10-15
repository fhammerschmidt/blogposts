In the world of JavaScript, many APIs are not designed in a way a functional developer might design them. Although it got a lot better over the last couple of years thanks to technologies like React, most browser APIs are still rather object-oriented or lower-level.

[**ReScript**](https://rescript-lang.org/) (previously BuckleScript) is a statically typed and sound language, compiler and build system, which generates very readable JavaScript. Thus, it is a great alternative to TypeScript for people who can't stand type holes.

**ReScript** offers a wide variety of decorators to make binding to JavaScript APIs easier.

Recently, I was required to use the browser's [`mediaDevices` API](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices). One of the most important methods is the [`getUserMedia`](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia) method which yields a Promise containing a `MediaStream` object. It also has one required parameter, the `constraints` object, to determine if the API user requests audio, video or both. For example, to only request an audio stream one would use the following JS code:

```js
navigator.mediaDevices.getUserMedia({ audio: true, video: false });
```

For video only you would set `audio: false, video: true` and only if you require both, `audio: true, video: true`.

In idiomatic **ReScript**, such an API would probably be designed in one of the following ways:

- Either via a `constraints` variant:

```reasonml
type constraints = Audio | Video | Both

let getUserMedia: constraints => Js.Promise.t<stream> = ...

/* Calling the function */
getUserMedia(Audio)
getUserMedia(Video)
getUserMedia(Both)
```

- Or three separate functions

```reasonml
let getUserAudio: unit => Js.Promise.t<stream> = ...
let getUserVideo: unit => Js.Promise.t<stream> = ...
let getUserMedia: unit => Js.Promise.t<stream> = ...

/* Calling the function */
getUserAudio()
getUserVideo()
getUserMedia()
```

In this case, I prefer the latter one. Soon you will see, why.

# Embracing externals

The default way to bind to existing JS functions is the external keyword accompanied by a bunch of different decorators/annotations, which always start with `@`.

> **NOTE**: With the most current bundle of the ReScript platform (bs-platform 8.3), it will be possible to [omit the `bs` part](<(https://rescript-lang.org/blog/release-8-3-pt1#lightweight-ffi-attributes-without-bs-prefix)>) in the annotations. So `@bs.as` will turn to just `@as`, etc. Check if your version is new enough for even cleaner code.

## Binding attempt 1

To bind to global values such as `navigator.mediaDevices.getUserMedia`, two annotations are necessary. Let's have a look at how one can write the binding and dissect it into parts later on.

```reasonml
/* Navigator.MediaDevices module */
type constraints = {
  audio: bool,
  video: bool,
}

@bs.val @bs.scope(("navigator", "mediaDevices"))
external _getUserMedia: constraints => Js.Promise.t<stream> =
  "getUserMedia"
```

1. `type constraints = ...`:
   The first part is a type definition for the constraints parameter we need to give to the `getUserMedia` function. It is a so-called record which looks like an object and also compiles to a JS object of the same shape, but is actually [something different](https://rescript-lang.org/docs/manual/latest/record).

2. `@bs.val` is to bind to global values. Global values are values which you always have in scope, like `navigator` or `window`. Well, depending on the target platform (browser, node, etc.) of course.

3. `@bs.scope(("navigator", "mediaDevices"))`: When those values are nested, you need to define its scope too with `@bs.scope`. Here, we use a tuple of strings to tell the compiler that the scope is multiple levels deep. You can tell that it's a tuple because of the double `((` and `))`. The outer parentheses are from the scope function, and the inner ones are from the tuple. Alternatively, you can also just use a string for the whole thing like this: `@bs.scope("navigator.mediaDevices")`.

4. `external _getUserMedia`: The aforementioned annotations only work together with the `external` keyword. With it you define the name of the binding, which does not necessarily need to be the same as the JavaScript method you want to bind to.

5. `: constraints => Js.Promise.t<stream>`: Then comes the type annotation. According to MDN, this takes the constraints object mentioned earlier and returns a Promise of MediaStream, which translates to the built-in **ReScript** type `Js.Promise.t`. The `<stream>` type which is wrapped by the promise is unfortunately not built-in, but can be typed accordingly (left as an exercise to the reader 😉).

6. `= "getUserMedia"`: At last, a string with the actual name of the function. Watch out for typos!

We also define a helper record here. It looks like an object and compiles to a JS object, but is actually something different.

Now we have a binding to `getUserMedia`, but do we really want to call that ugly `Navigator.MediaDevices._getUserMedia({audio: true, video: true}` everywhere in the codebase, or can we achieve a nicer API?

Let's write some helper functions:

```reasonml
/* Navigator.MediaDevices module continued */
let getUserAudio = () => _getUserMedia({audio: true, video: false})
let getUserVideo = () => _getUserMedia({audio: false, video: true})
let getUserMedia = () => _getUserMedia({audio: true, video: true})
```

The functions can then be called from anywhere in the codebase via:

```reasonml
Navigator.MediaDevices.getUserAudio()
Navigator.MediaDevices.getUserVideo()
Navigator.MediaDevices.getUserMedia()
```

> **NOTE**: Simplified example, in practice you would use `Js.Promise.then_` or the like, to retrieve the stream itself.

Yay, that looks better. But now we generate some extra code
like this function:

```js
function getUserAudio(param) {
  return navigator.mediaDevices.getUserMedia({
    audio: true,
    video: false,
  });
}
```

## Binding attempt 2

Can we do better and still populate the functions with default parameters? I'd say yes, with this nifty trick.

There is also the very helpful `@bs.as` annotation for when you need to give an entity a complex name which you could otherwise not create. For instance, in record fields, there is no `-` allowed which can be mitigated by this annotation.

`@bs.as` takes a single string as an argument. Coincidentally, JSON is basically just a complex string and thus we can inject some complex objects into the default configuration, too.

Just write

```
@bs.as(json`{your-config-object}`)
```

with a `_` afterwards to omit the value when calling the function.
Also we add a final `unit` (the only "parameter" of the function).

```reasonml
/* Navigator.MediaDevices module */
@bs.val @bs.scope(("navigator", "mediaDevices"))
external getUserAudio: (
  @bs.as(json`{"audio": false, "video": true}`) _,
  unit,
) => Js.Promise.t<stream> = "getUserMedia"
```

The **as-JSON trick** as I call it has its limits, though. It only works for JSON-compliant values, such as arrays, objects, strings, numbers and booleans. Functions would not work for instance.

We can do the same with the other two possible functions:

```reasonml
/* Navigator.MediaDevices module continued */
@bs.val @bs.scope(("navigator", "mediaDevices"))
external getUserVideo: (
  @bs.as(json`{"audio": true, "video": false}`) _,
  unit,
) => Js.Promise.t<stream> = "getUserMedia"

@bs.val @bs.scope(("navigator", "mediaDevices"))
external getUserMedia: (
  @bs.as(json`{"audio": true, "video": true}`) _,
  unit,
) => Js.Promise.t<stream> = "getUserMedia"
```

And end up with the same API as in attempt 1. Again, we can call our functions the same way as before:

```reasonml
Navigator.MediaDevices.getUserAudio()
Navigator.MediaDevices.getUserVideo()
Navigator.MediaDevices.getUserMedia()
```

But this time with no build artifacts, the second attempt comes with **zero cost** 🎉.

**Rule-of-thumb**: `let`s generate additional JS code, `external`s don't.

Thank you for reading and I hope this trick helps some of you. For a condensed example, please have a look at the [ReScript playground example](https://tinyurl.com/y3owk33j) this blog post is based on.
