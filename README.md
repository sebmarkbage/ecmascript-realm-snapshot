Realm Snapshot for ECMAScript
-----------------------------

An API that allows an ECMAScript program to have its heap and parsed functions serialized into an opaque VM specific format. This can then be stored in caches, transferred between workers and restored for fast start-up times.

This proposal is similar to the [Realms](https://github.com/tc39/proposal-realms) and [Frozen Realms](https://github.com/tc39/proposal-frozen-realms) proposals but solves a different use case, with a separate API. It can be proposed and progress independently.

## Motivation

ECMAScript programs typically have an initialization phase. These programs execute scripts that initializes some classes, functions, modules, helper objects etc. Only after that is done can they start running the actual program. This initialization may also include temporary objects required to wire up initialization which can thrash memory usage early in the start up path. Additionally, parsing and compiling the source of these programs takes significant time. Fast initialization time is critical for both local tools and web sites. It is currently an aspect where JavaScript is falling behind other languages/systems.

It [has been shown](http://blog.atom.io/2017/04/18/improving-startup-time.html) that using [V8's heap snapshot](https://v8project.blogspot.com/2015/09/custom-startup-snapshots.html) you can significantly improve start up time by starting from a pre-serialized heap. Separately, preparsing and precompilation of scripts have also proven improve start up time.

## Goals

### MVP

It is an explicit goal of this proposal to allow implementations to use a wide variety of implementation details to create the snapshot. This may include simply storing the source code and reevaluating it in order. This makes it possible for any implementation to support some form of this API. It also makes it possible to undo any optimizations this API provides if they're no longer needed. E.g. if a browser wants to throttle AOT compilation. Therefore, it is a goal to allow the engine to defer execution of any compiled scripts. Async compilation should be possible.

It's a goal to be able to preserve any optimizations that are associated with a Realm (or larger isolates/heaps that may contain multiple Realms).

It is a goal to be able to share a heap/isolate/vm with other Realms that were created outside of a snapshot. E.g. it should be possible to restore a Realm snapshot into a Web Browser's main thread. Not just into isolated Workers. _NOTE: This might be difficult to implement in current VM architectures._

It's a goal to provide ways to warm up a snapshot without having to serialize the resulting objects. E.g. to cause parsing/codegen of lazy functions and warm hidden class caches.

It's a goal to design an API that doesn't encourage storing results on the global object of the Realm.

It is a goal to provide a plausible path forward to a declarative Web API for loading precompiled snapshots.

### Non-Goals in the MVP

It is a non-goal to provide a declarative loading API for the Web as part of this proposal but this proposal should take such a future hook into account. This is similar to [WebAssembly's path](https://github.com/WebAssembly/design/issues/972).

It's a non-goal to support partial Realms (such as individual objects) to be serialized independently in this MVP but the API should support expanding to that in the future.

It's a non-goal to support ECMAScript and WebAssembly to be compiled into a single snapshot for now.

It's a non-goal to support any form of abstraction interpretation and serialization of placeholder functions or values.

It's a non-goal to override the global proxy and provide alternative intrinsics in the MVP but that can be added later.

## Proposed API

`RealmBuilder(hooks)` - Constructor to create an opaque Realm which can be compiled as a snapshot. Accepts an optional object with hooks similar to the [Realm API](https://github.com/tc39/proposal-realms). In the initial proposal none of the hooks are supported though. (`importHook` could be supported but it's still controversial so it's pulled from this proposal until that's resolved.) Returns an exotic object that as a fresh new Realm stored in its `[[Realm]]` slot. This realm's global object is a simple global object with no host APIs. Same as `new Realm().global`.

`RealmBuilder.prototype.evalScript(scriptSource)` - Evaluates a script source in the global scope of the Realm stored in the RealmBuilder's `[[Realm]]` slot. Returns a new `RealmValue` which stores the completion value. If the script threw that's stored in the `RealmValue` as well. In the MVP there is no way for the script to communicate with the outside world so execution doesn't actually have to happen here. It could be deferred to compilation or instantiation.

`RealmBuilder.prototype.evalModule(specifier)` - Loads a module specifier using the `[[ImportHook]]` and its dependencies. Then executes it with in the Realm stored in the RealmBuilder's `[[Realm]]` slot. If initialization throws, that's stored in the returned `RealmValue`. If not, the returned `RealmValue` stores the `Module`.

`RealmValue` - Constructor for the opaque values returned by `evalScript` and `evalModule`. Throws if invoked directly. It has an internal slot representing completion values.

`RealmSnapshot` - Constructor for the opaque result returned by `compile`. Throws if invoked directly. It has an internal slot representing a compiled buffer which can be accessed by Host environment APIs in other specs.

`RealmSnapshot.compile(realmBuilder, realmArguments)` - Accepts an instance of a `RealmBuilder` and an optional `Iterable` of `RealmValue` instances. Returns a `Promise<RealmSnapshot>`. Each `RealmValue` has to be associated with this particular `RealmBuilder`'s environment, otherwise the promise is rejected. If compilation is successful, the Promise resolves into a `RealmSnapshot` object with its internal slot containing the compiled buffer representing the state of the `realmBuilder`'s internal `[[Realm]]`, its reachable references and those of each `RealmValue`.

`RealmSnapshot.instantiate(realmSnapshot)` - Accepts a `RealmSnapshot` and returns a freshly instantiated `Realm` that is the equivalent of replaying the scripts that was issued on the `RealmBuilder`. The returned `Realm` has an extra property called `arguments` which contains the normal values that was passed as the second argument to `compile` (note that these are no longer opaque `RealmValue`).

`RealmSnapshot.instantiate(realmBuilder, realmArguments)` - Overloaded method. Shorthand for `RealmSnapshot.instantiate(await RealmSnapshot.compile(realmBuilder, realmArguments))`.

### Nested Realms

It's possible to create `RealmSnapshot` and normal `Realm` objects from within the `RealmBuilder`'s environment and have them be serialized as well. Therefore it is possible to serialize multiple realms into a single snapshot.

### Restricted APIs

There are two APIs in ECMAScript that won't yield deterministic results when reexecuted. `Math.random()` and `Date.now()` / `new Date()`. In the MVP these could be restricted inside the RealmSnapshot environment (e.g. they throw). In the future we could add hooks that allows the running realm to configure the values that are returned.

### Possible Future APIs

In the future we might enable `Date.now` and `Math.random` to be simulated using hooks. We could also expose the ability to expose custom environment hooks in the environment. E.g. to simulate a DOM environment. We could also expose templates/placeholders for functions that, when residual in the snapshot, are replaced with custom functions upon restore.

A RealmSnapshot could be frozen to create a [Frozen Realm](https://github.com/tc39/proposal-frozen-realms) that can then be cheaply restored and spawn many new realms.

We could in the future add an API that creates one main start up snapshot that contains strong references from the global Realm, and several partial snapshots that can be restored individually on top of the start up snapshot.

## Application Examples

The RealmSnapshot API itself doesn't provide access to the serialized form. The Host environment provides that through a serialization and deserialization API. The following are some examples of how those APIs could work in those environments.

### Web: Structured Clone of a RealmSnapshot

In a Web host `RealmSnapshot` is a cloneable object. It can be cloned between windows/workers. It can also be stored into and retrieved from an `IDBObjectStore`. The semantics o a structured clone is as if the same source was evaled into as-if the binary source, from which the `RealmSnapshot` was compiled, were cloned and recompiled into the target realm. Engines should attempt to share/reuse internal compiled code when performing a structured clone although, in corner cases like CPU upgrade or browser update, this may not be possible and full recompilation may be necessary.

Given the above engine optimizations, structured cloning provides developers explicit control over both compiled-code caching and cross-window/worker code sharing.

Example:

```js
let realmBuilder = new RealmBuilder();
let realmValue = realmBuilder.evalScript(scriptSource);
let snapshot = await RealmSnapshot.compile(realmBuilder, [realmValue]);
indexedDBStore.put(snapshot, "cache");
```

```js
let request = indexedDBStore.get("cache");
request.onsuccess = async (event) => {
  let snapshot = request.result;
  let realm = await RealmSnapshot.instantiate(snapshot);
  let [realmValue] = realm.arguments;
  ...
};
```

### Web: Automatic Caching of a Realm

Typical use case for loading a snapshot involves a complex loading strategy and caching mechanism. Browser engines [prefer to control that whole pipeline](https://github.com/WebAssembly/design/issues/972) at least  as a default. There could also be a declarative API to load a pre-compiled Realm that lets the browser do its own caching.

WebAssembly settled on [an extension API for Web Embedding](https://github.com/WebAssembly/design/blob/master/Web.md#additional-web-embedding-api) where `compile` and `instantiate` has additional forms that accept a `Response` object to be passed in directly.

Since the `RealmBuilder` can consist of several scripts we'll need something more granular than that. One possible solution is to extend `RealmBuilder` with streaming variants of `eval`.

`RealmBuilder.prototype.evalStreamingScript(response)` - Accepts a `Response` object and returns a `RealmValue`. This `RealmValue` is a lazy value. When it is passed to `compile`, then `compile` must first resolve the `.text()` Promise of the response and pass that to `evalScript`. If that Promise is rejected, then the Promise returned by `compile` is rejected.

`RealmBuilder.prototype.evalStreamingModule(response)` - Same as `evalStreamingScript` but uses `evalModule` underneath.

Example:

```js
// Loading an auto-precompiled and cached Realm
let response = await fetch(url);
let realmBuilder = new RealmBuilder();
let realmModule = realmBuilder.evalStreamingModule(response);
let realm = await RealmSnapshot.instantiate(realmBuilder, [realmModule]);
let [moduleInstance] = realm.arguments;
```

### Node: Buffer from a RealmSnapshot

Node environments are not concerned about leaking implementation details of host environments. The VM module already exposes [V8 cache data for parsing and code gen](https://nodejs.org/api/vm.html#vm_new_vm_script_code_options). Therefore, the raw binary representation can be exposed and stored in arbitrary places by the VM module.

Example:

```js
import { writeFileSync } from "fs";
import { bufferFromSnapshot } from "vm";

let realmBuilder = new RealmBuilder();
let realmValue = realmBuilder.evalScript(scriptSource);
let snapshot = await RealmSnapshot.compile(realmBuilder, [realmValue]);
writeFileSync("cache", bufferFromSnapshot(snapshot));
```

```js
import { readFileSync } from "fs";
import { snapshotFromBuffer } from "vm";

let snapshot = snapshotFromBuffer(readFileSync("cache"));
let realm = await RealmSnapshot.instantiate(snapshot);
let [realmValue] = realm.arguments;
```

## FAQ

### Will This Be Compatibile with the JS Ecosystem?

One key question is whether people will actually be able to use this API with the limitations that it provides. There is [some precedent](http://blog.atom.io/2017/04/18/improving-startup-time.html) that if the benefits are important enough larger central frameworks are willing to do the work to reorganize their code in such a way that it is possible. After this architecture is in place in key aspects, smaller libraries follows.

However, it is also possible that [smart compilers](https://prepack.io/) are able to take existing code filled with abstract dependencies and then rewrite it to a two phase initialization model. The first phase initializes a the pure heap and the second phases patches it up. This provides an upgrade path for the existing ecosystem to be able to adopt this feature. Then rewrite it in such a way that a compiler is not needed.

### Why TC39 Instead of Web Standards?

The same technique is needed by many other environments such as Node CLIs, Electron apps, React Native, etc.

Not all scripts can be automatically supported as a snapshot. The way libraries get organized on shared networks, like npm, will highly depend on the details of the snapshotting mechanism. The ecosystem effects of a consistent set of features and non-features is important.

Realms are a complicated part of the ECMAScript spec. It's important that TC39 is aware of how all these pieces fit together and how they may affect snapshots as well as how they relate [to](https://github.com/tc39/proposal-weakrefs) [other](https://github.com/tc39/proposal-realms) [proposals](https://github.com/tc39/proposal-frozen-realms).

JavaScript VMs are typically built as separate projects from Web Standards in a Browser (and reused across environments). This proposal is expected to be highly coupled to the VM implementation details and almost fully involve work from the VM teams. By putting the specification under TC39 it naturally follows the same test suites and specifications that these teams are already most concerned with.

That said, this all ties into the whole strategy of script loading in a Browser environment and this will involve stake holders across all levels. We're simply start the discussion in the venue of TC39.

## Why Constrain the API to Implementation Flexibility?

This proposal is inherently implementation driven due to performance concerns. A goal of this proposal is to provide the semantic specifications that makes more types of optimizations possible, not to constrain them. However, it is also important to note that this should also provide __enough__ constraints to make optimizations possible at all.

There are many possible architectural designs of a VM. We could litigate the legitimacy of any particular VM for ever. It is important to be able to allow a lot of flexibility and have different strategies compete. Therefore it is important that this proposal welcomes feedback from all implementors about how this might affect their implementation concerns.

### Why Not Just Preparse Scripts (like the Function constructor)?

An alternative or complementary proposal could be to only do explicit preparsing/precompilation of script or function bodies. Similar to the Function constructor. That approach is also helpful but much more limited in the types of optimizations that a VM can choose to do. E.g. It doesn't allow information about hidden classes and expected object signatures to map to preserialized objects. Meanwhile, serialized Realms can contain individual functions as their exports so it also fully supports that use case. A VM can choose to only store preparsing of the script content.

### Why Not Just Use The Information From Previous Runs?

One optimization browsers already use is caching code generated from a previous visit to a page. Since this code has already executed once it is possible to cache even optimization information, inline caches, hidden classes etc. This helps to some extent but it doesn't allow a heap to be safely serialized. It must reexecute every time.

Additionally, this doesn't let this code get optimistically pre-executed. In an offline first loading model, provided by Service Workers, you typically run from the previous version of your code and upgrade to a newer version in the background. Frequently revisits will hit a fresh new version every time they visit the page. Downloading scripts can be done before first visit. Some preparsing can be done. However, optimizations that can be done __before__ first execution is limited. It is not safe to pre-execute arbitrary Realms. These Realm Snapshots provide a safe environment for code to be pre-executed.

### Why Not Serialize Any Existing Object Graph (like JSON)?

Functions in this graph will have a Realm associated with them. It is also possible for other objects to have Realms associated with them in the internal representations. There might also be multiple Realms in this chain. Saving and restoring this information adds a lot of complexity to the API and the implementation.

A VM might want to use a different representation for allocating objects for snapshot purposes. E.g. it might want to use a separate heap so that the heap can cheaply be serialized as is rather than scanned and recreated. A VM may also want to use a representation outside of the normal JS heap for normal runtime but keep it on the JS heap for snapshot purposes.

The heap is expected to contain closures and private state. Allowing arbitary objects to be serialized would open up the ability to probe the content of them. It wouldn't be completely open since the snapshot representation is opaque but clever hacks might be able to use it as a new attack surface.

### How Does This Relate to AMP and HTML Import Caching?

This is complementary. It's a smaller piece of a bigger platform that can safely precache larger pages. It starts smaller by making more pieces of the system incrementally deterministic. E.g. a Google AMP precompiler can start supporting pre-executing Realm Snapshot imports without necessarily making all scripts required to be deterministic.

### Prior ART in the JS Ecosystem

[V8](https://v8project.blogspot.com/2015/09/custom-startup-snapshots.html)

[Atom](http://blog.atom.io/2017/04/18/improving-startup-time.html)

[Prepack](https://prepack.io/)

[Moddable](https://github.com/rwaldron/tc39-notes/blob/master/es8/2017-05/may-24.md#14iii)

[WebAssembly](http://webassembly.org/docs/js/#webassemblycompile)

### Related Proposals

[Realms](https://github.com/tc39/proposal-realms)

[Frozen Realms](https://github.com/tc39/proposal-frozen-realms)

## [Status of this Proposal](https://github.com/tc39/ecma262)

This has not yet been presented to TC39. I'm still gathering feedback.
