# svelte-hmr

HMR commons for Svelte 3.

This packages provides shared dependencies for implementing Svelte HMR in any bundler plugins.

If you want to _use_ HMR in your Svelte project, what you need is a HMR enabled plugin for your bundler (e.g. Rollup or Webpack). You'll find a list of available tools at the end of this doc.

On the other hand, if you are really developing a plugin... Sorry, no docs for now! Drop me a line, I'd be happy to help!

## Features

- update Svelte components in place

- preservation of component state, including local state (i.e. `let` vars in your components)

- inject CSS instead of doing a full replace when only the component's CSS has changed, with compatible HMR APIs (`rollup-plugin-hot` and Nollup for now)

## What's HMR, by the way?

> **NOTE** To avoid repetition, the following text only mentions HMR in the context of browsers, but it can also be used in other platforms. For example `svelte-hmr` is also used in Svelte Native.

HMR stands for Hot Module Replacement. It is a tool that is used during development to replace only the parts that have changed in a _running_ application, without the need to reload the whole browser page.

It's nice because it shortens your feedback loop (you don't lose the current state of the page you're working it with each code change), and it feels like magic! :sparkles:

Well, since you're reading this, let me tell you a little more about HMR. Magic is actually not such a good think in software development, so if we can demystify HMR a bit, it will probably benefits you when it comes to answer setup questions or, generally, get the most out of your HMR experience.

So... There are multiple layers to HMR. The first one is the technical capacity to actually replace a JS _module_ (think ES module -- in practice a JS file) at runtime. This capacity is provided (or not) by your bundler or dev server (e.g. Webpack, Parcel, Vite, Snowpack...). A notable absent in the HMR capable bundlers is Rollup, but a HMR plugin exists, as well as a Rollup-compatible super fast dev bundler and server, Nollup (see links below).

The HMR capacity essentially revolves around watching your file system and sending events to the browser when this happens (and, of course, doing all the other bundler stuff, but they're not interesting for this discussion).

In the browser, we need something to receive and process those events. And so there is a HMR runtime (i.e. some JS code) that is injected in the browser when you enable HMR. This runtime exposes a HMR (or hot) API. The hot API differs with each bundler, even if there is some level of standardization. For example, `rollup-plugin-hot`'s API looks like this:

```js
if (imp‎ort.meta.hot) {
  import.meta.hot.dispose(data => {
    // do cleanup
  })
  import.meta.hot.accept(() => {
    // adjust side effects
  })
}
```

Why do we need a hot API?

Because just replacing a whole chunk of JS code doesn't really gets you what you want. Let's think about it: what does replacing a JS module really means? You can't unrun the previously executed module. So that's merely running the new version of the module.

And that's about all what your HMR server will give you. But it's not enough. If your module was a helper function, new calls will go to the new version of the function, but the result of calling the previous version are still visible on the page. Same with a UI component: newly created components will be fresh, but existing components on the page are still from the previous versions...

That's why we need this hot API. We need to provide HMR handlers, that is HMR specific code to properly apply the effects of the code change in the app, and cleanup the effects of the previous module versions.

Actually, what should be done to reflect a code update is hyper specific to the content of the module. Typically, you'd need specific HMR handlers for each one of your modules (i.e. files). For most kind of contents, this cannot be automatized. Oh, and did I tell you that HMR handlers are pretty tedious things to write?

Oops. Does this makes HMR essentially worthless? No! Because, fortunately, there is one kind of content for which we know how to do automatic HMR handlers. Those are UI components. When a HMR update hits a component, we can destroy all existing instances of the component and replace them with new instances created with the updated version of the component. Bim. Well, there's also a part of dark magic involved because we need to recreate the new instances in the same state as the previous instance we're destroying. We need to pass the right props, transfer inner state, local state, event listeners, etc.

This kind of automatic HMR handlers for Svelte components -- and associated dark magic -- is what `svelte-hmr` provides! (For the sake of completeness, it also provides some Node utils to help bundlers apply the code transform needed to inject those HMR handlers into a Svelte component.)

Cool! So you can have HMR with a lot of your modules without having to write HMR handlers yourself, that's good news. And there are more. HMR APIs implements a _bubbling mechanism_. In HMR linguo, when a module has some HMR handlers, it is said to be "accepted". When a HMR update hits a module that is not accepted, this will not immediately translate into a full reload of the page. Instead, the HMR engine will check all the modules that imports the unaccepted module and, if they are all accepted, then it will refresh all those parent modules instead (refresh means running HMR handlers, here). If some of the parents are themselves unaccepted, the HMR update will continue to bubble up toward the app's entry point, stopping at accepted module on each branch. If an update hits the entry point (and the entry point itself is not accepted), then only will it command a full reload.

It makes sense: all those modules boast of being able to apply a code change to themselves so, if we refresh all of them, we should theoretically also see the updated effect of their dependencies, no? Well, unfortunately, it depends... If your updated module is exporting values or pure functions, it will work perfectly. If your module uses `setInterval` and there isn't a HMR handler to dispose that, the `setInterval` function will keep running (and each HMR update will probably add a new one). It all depends on how stateful or how much side effects your module has. But in many cases it can work and, in those cases, you'll also get HMR for free (since all HMR updates will probably be stopped by Svelte components). For the remaining cases... Well, you're probably left with [learning how to write HMR handlers](https://github.com/rixo/rollup-plugin-hot#api), or accept to manually reload the browser.

Here you are. This is what HMR is, and how it works.

I think the most important take away is that the component that is affected by a change will be recreated. Its state will be preserved but all its children will be recreated too, and their state not preserved. This is a necessity because the elements / child components structure of a component can be entirely different after the file has been modified (and we don't really have the technical capacity to track children).

Now, the best way to see what it can do for you is probably to checkout the template bellow and get your hands at it! (Add 500 components and try Nollup, you should love the speed!)

### Preservation of local state

Local state is preserved by Svelte HMR, that is any state that Svelte itself tracks as reactive (basically any root scope `let` vars, exported or not).

This means that in code like this:

~~~svelte
<script>
  let x = 1
  x++ // x is now 2
</script>

<p>{x}</p>
~~~

If you replace `let x = 1` by `let x = 10` and save, the previous value of `x` will be preserved. That is, `x` will be 2 and not 10. The restoration of previous state happens _after_ the init code of the component has run, so the value will not be 11 either, despite the `x++` that is still here.

If you find this behaviour inconvenient, you can disable it by setting `noPreserveState` option of your HMR-enabled bundler plugin.

If you want to disable it for just one particular file, or just temporarily, you can turn it off by adding a `// @!hmr` comment somewhere in your component.

## Svelte HMR tools

### Rollup / Nollup

- [svelte-template-hot] :arrow_left: **Recommended starting point**
- [rollup-plugin-svelte-hot]

#### HMR support for Rollup

Rollup does not natively support HMR. You'll need to use one of the following solutions. The best way to get started is to refer to [svelte-template-hot], that demonstrates usage of both.

- [rollup-plugin-hot]
- [Nollup][nollup]

### Webpack

Documentation & examples for Webpack are currently lagging behind...

- [svelte-loader-hot]
- [svelte-template-webpack-hot]

### Sapper

Sapper can be supported with Webpack's loader. The link bellow is still very much a work in progress (no docs at time of writing), but I'm adding it for future reference.

- [sapper-template-hot#webpack](https://github.com/rixo/sapper-template-hot#webpack)

Some initial work has also been made on supporting Sapper with Rollup, and basic support for simple cases is available. But this one is still in very early stages (and, again, poorly documented for now, sorry). I could really use some help with this one actually, if you're in the mood ;)

- [sapper-template-hot#rollup](https://github.com/rixo/sapper-template-hot#rollup)

### Svelte Native

The official Svelte Native template already includes HMR support.

- [svelte-native-template]

### Vite

- [vite-plugin-svelte]
- [svite]

[nollup]: https://github.com/PepsRyuu/nollup
[rollup-plugin-hot]: https://github.com/rixo/rollup-plugin-hot
[rollup-plugin-svelte-hot]: https://github.com/rixo/rollup-plugin-svelte-hot
[rollup-plugin-svelte]: https://github.com/rollup/rollup-plugin-svelte
[svelte-template-hot]: https://github.com/rixo/svelte-template-hot
[svelte-template]: https://github.com/sveltejs/template
[svelte-native-template]: https://github.com/halfnelson/svelte-native-template
[svelte-loader-hot]: https://github.com/rixo/svelte-loader-hot
[svelte-template-webpack-hot]: https://github.com/rixo/svelte-template-webpack-hot
[vite-plugin-svelte]: https://github.com/intrnl/vite-plugin-svelte
[svite]: https://github.com/dominikg/svite

## License

[ISC](LICENSE)
