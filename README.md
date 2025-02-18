# @plotdb/setimmediate

fork of `setImmediate.js` by `YuzuJS`, patching for rescope environment.

instead of `npm install --save setimmediate`, use this:

    npm install --save @plotdb/setimmediate



## Why

Original setImmediate.js checks `global` against `event.source` (line 112, setImmediate.js). However, `@plotdb/rescope` replace global with an proxied object - which makes this comparison fail, and thus, callback functions passed to `setImmediate` will never be run.

Generally speaking, this is a limitation of `@plotdb/rescope` - Proxy object should not be the same with the proxied object, and we probably should not mess up with the event emitter about event's source.

Since `setImmediate.js` checks the existence of `setImmediate` before initialization, we can simply provide an alternative version such as this `@plotdb/setimmediate` to workaround this issue for now - although this may increase the required JS file size by about 1.6K.


----

# setImmediate.js
**A YuzuJS production**

## Introduction

**setImmediate.js** is a highly cross-browser implementation of the `setImmediate` and `clearImmediate` APIs, [proposed][spec] by Microsoft to the Web Performance Working Group. `setImmediate` allows scripts to yield to the browser, executing a given operation asynchronously, in a manner that is typically more efficient and consumes less power than the usual `setTimeout(..., 0)` pattern.

setImmediate.js runs at “full speed” in the following browsers and environments, using various clever tricks:

 * Internet Explorer 6+
 * Firefox 3+
 * WebKit
 * Opera 9.5+
 * Node.js
 * Web workers in browsers that support `MessageChannel`, which I can't find solid info on.

In all other browsers we fall back to using `setTimeout`, so it's always safe to use.

## Macrotasks and Microtasks

The `setImmediate` API, as specified, gives you access to the environment's [task queue][], sometimes known as its "macrotask" queue. This is crucially different from the [microtask queue][] used by web features such as `MutationObserver`, language features such as promises and `Object.observe`, and Node.js features such as `process.nextTick`. Each go-around of the macrotask queue yields back to the event loop once all queued tasks have been processed, even if the macrotask itself queued more macrotasks. Whereas, the microtask queue will continue executing any queued microtasks until it is exhausted.

In practice, what this means is that if you call `setImmediate` inside of another task queued with `setImmediate`, you will yield back to the event loop and any I/O or rendering tasks that need to take place between those calls, instead of executing the queued task as soon as possible.

If you are looking specifically to yield as part of a render loop, consider using [`requestAnimationFrame`][raf]; if you are looking solely for the control-flow ordering effects, use a microtask solution such as [asap][].

## The Tricks

### `process.nextTick`

In Node.js versions below 0.9, `setImmediate` is not available, but [`process.nextTick`][nextTick] is—and in those versions, `process.nextTick` uses macrotask semantics. So, we use it to shim support for a global `setImmediate`.

In Node.js 0.9 and above, `process.nextTick` moved to microtask semantics, but `setImmediate` was introduced with macrotask semantics, so there's no need to polyfill anything.

Note that we check for *actual* Node.js environments, not emulated ones like those produced by browserify or similar. Such emulated environments often already include a `process.nextTick` shim that's not as browser-compatible as setImmediate.js.

### `postMessage`

In Firefox 3+, Internet Explorer 9+, all modern WebKit browsers, and Opera 9.5+, [`postMessage`][postMessage] is available and provides a good way to queue tasks on the event loop. It's quite the abuse, using a cross-document messaging protocol within the same document simply to get access to the event loop task queue, but until there are native implementations, this is the best option.

Note that Internet Explorer 8 includes a synchronous version of `postMessage`. We detect this, or any other such synchronous implementation, and fall back to another trick.

### `MessageChannel`

Unfortunately, `postMessage` has completely different semantics inside web workers, and so cannot be used there. So we turn to [`MessageChannel`][MessageChannel], which has worse browser support, but does work inside a web worker.

### `<script> onreadystatechange`

For our last trick, we pull something out to make things fast in Internet Explorer versions 6 through 8: namely, creating a `<script>` element and firing our calls in its `onreadystatechange` event. This does execute in a future turn of the event loop, and is also faster than `setTimeout(…, 0)`, so hey, why not?

## Usage

In the browser, include it with a `<script>` tag; pretty simple.

In Node.js, do

```
npm install --save setimmediate
```

then

```js
require("setimmediate");  // (somewhere early in your app; it attaches to the global scope.)
```


## Demo

* [Quick sort demo][cross-browser-demo]

## Reference and Reading

 * [Efficient Script Yielding W3C Editor's Draft][spec]
 * [W3C mailing list post introducing the specification][list-post]
 * [IE Test Drive demo][ie-demo]
 * [Introductory blog post by Nicholas C. Zakas][ncz]


[spec]: https://dvcs.w3.org/hg/webperf/raw-file/tip/specs/setImmediate/Overview.html
[task queue]: http://www.whatwg.org/specs/web-apps/current-work/multipage/webappapis.html#task-queue
[microtask queue]: http://www.whatwg.org/specs/web-apps/current-work/multipage/webappapis.html#perform-a-microtask-checkpoint
[raf]: https://html.spec.whatwg.org/multipage/webappapis.html#dom-window-requestanimationframe
[asap]: https://github.com/kriskowal/asap
[list-post]: http://lists.w3.org/Archives/Public/public-web-perf/2011Jun/0100.html
[ie-demo]: http://ie.microsoft.com/testdrive/Performance/setImmediateSorting/Default.html
[ncz]: http://www.nczonline.net/blog/2011/09/19/script-yielding-with-setimmediate/
[nextTick]: http://nodejs.org/docs/v0.8.16/api/process.html#process_process_nexttick_callback
[postMessage]: http://www.whatwg.org/specs/web-apps/current-work/multipage/web-messaging.html#posting-messages
[MessageChannel]: http://www.whatwg.org/specs/web-apps/current-work/multipage/web-messaging.html#channel-messaging
[cross-browser-demo]: http://jphpsf.github.com/setImmediate-shim-demo
