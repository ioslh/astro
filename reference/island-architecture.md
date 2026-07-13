# Astro Island Architecture

## Overview

Astro's island architecture has two main execution paths:

1. **Client islands (`client:*`)**: Astro renders static HTML on the server, wraps only interactive components in `<astro-island>`, and hydrates them later in the browser.
2. **Server islands (`server:defer`)**: Astro renders fallback content first, then fetches the island's HTML separately and replaces the placeholder.

At a high level, the implementation works like this:

1. The compiler marks components that should become islands
2. SSR extracts hydration directives and emits `<astro-island>`
3. Astro injects the island runtime and client directive scripts into the page
4. The browser runtime loads and hydrates islands according to their `client:*` strategy
5. `server:defer` uses a dedicated endpoint to return island HTML

---

## 1. Compile time: identifying island components

Key files:

- `packages/astro/src/vite-plugin-astro/index.ts`
- `packages/astro/src/vite-plugin-astro/metadata.ts`

After compiling an `.astro` file, `vite-plugin-astro` stores these results in `meta.astro`:

- `hydratedComponents`
- `clientOnlyComponents`
- `serverComponents`

That metadata tells the rest of the pipeline:

- which components need client hydration
- which components are `client:only`
- which components are server islands

This is the foundation for Astro's later build, SSR, and runtime behavior.

---

## 2. SSR: extracting `client:*` directives

Key file:

- `packages/astro/src/runtime/server/hydration.ts`

`extractDirectives()` removes Astro's special props from the component input, including:

- `client:load`
- `client:idle`
- `client:visible`
- `client:media`
- `client:only`
- `client:component-path`
- `client:component-export`

The result is split into two parts:

1. **Regular props**: still passed into the server render
2. **Hydration metadata**: directive name, directive value, component module path, and export name

So Astro separates island-specific metadata from normal component props before rendering.

---

## 3. SSR: wrapping interactive output in `<astro-island>`

Key file:

- `packages/astro/src/runtime/server/render/component.ts`

This is the core server-side entry point for client islands.

The main flow is:

1. Render the component to static HTML through the matched renderer
2. If there is no `client:*` directive, emit the HTML directly
3. If there is a `client:*` directive, build an island description
4. Emit the component as an `<astro-island>` custom element

Important details:

- the result of `extractDirectives()` is copied into component metadata
- `generateHydrateScript()` serializes hydration data onto the island element
- the SSR HTML is preserved as the island's children
- slot content may be wrapped so it can be recovered during hydration

Astro's "partial hydration" is therefore implemented by isolating interactive regions, not by handing the whole page to a client framework.

---

## 4. Generating hydration metadata

Key file:

- `packages/astro/src/runtime/server/hydration.ts`

`generateHydrateScript()` adds the attributes the browser runtime needs, including:

- `component-url`
- `component-export`
- `renderer-url`
- `props`
- `client`
- `ssr`
- `opts`
- `before-hydration-url`

Those attributes carry:

- the component module URL
- the export name to read from that module
- the client entry for the framework renderer
- serialized props
- the hydration strategy (`load`, `idle`, `visible`, and so on)
- a marker that the island is still in SSR state
- directive arguments
- an optional script that must run before hydration

The browser runtime does not need a separate lookup step. It can reconstruct the island entirely from the `<astro-island>` attributes.

---

## 5. Injecting the runtime and directive scripts

Key files:

- `packages/astro/src/runtime/server/scripts.ts`
- `packages/astro/src/core/client-directive/default.ts`

During SSR, Astro injects two kinds of scripts as needed:

1. **Directive scripts**: define the trigger behavior for `client:load`, `client:idle`, `client:visible`, `client:media`, and `client:only`
2. **Island runtime script**: defines the `<astro-island>` custom element

The built-in directive registry lives in:

- `packages/astro/src/core/client-directive/default.ts`

The default built-in client directives are:

- `idle`
- `load`
- `media`
- `only`
- `visible`

Each one is injected from a prebuilt script and exposed to the browser runtime through `Astro[directive]`.

---

## 6. Browser runtime: the `astro-island` custom element

Key file:

- `packages/astro/src/runtime/server/astro-island.ts`

This is the most important runtime file for Astro islands.

It is responsible for:

1. defining the `astro-island` custom element
2. starting hydration when the element is connected to the DOM
3. reading the component URL, renderer URL, props, slots, and client directive from element attributes
4. dynamically importing the component module and renderer
5. reviving serialized props
6. calling the renderer hydrator to activate the component

There are a few especially important mechanisms here.

### `connectedCallback()` and `await-children`

During streaming SSR, the island element may be attached before all of its children arrive.

To handle that, the runtime can:

- watch child mutations
- wait for the `<!--astro:end-->` marker
- continue only after the island's children are complete

That prevents hydration from starting before the island HTML is fully present.

### `start()`

`start()` reads the island's `client` attribute and calls the matching directive:

- `load`
- `idle`
- `visible`
- `media`
- `only`

If the directive script has not registered yet, the runtime waits for `astro:${directive}` and retries.

### `hydrate()`

The actual activation happens in `hydrate()`:

- restore slot content
- revive serialized props
- call the framework hydrator

After hydration finishes, the runtime:

- removes the `ssr` attribute
- dispatches `astro:hydrate`

### Parent-before-child hydration

The runtime checks:

- `this.parentElement?.closest('astro-island[ssr]')`

If an ancestor island is still waiting to hydrate, the child waits until the parent emits `astro:hydrate`.

So Astro explicitly hydrates:

- **parent islands first**
- **child islands second**

That avoids nested islands being recreated or broken by a parent hydration pass.

---

## 7. `client:*` directives only decide *when* hydration happens

Key files:

- `packages/astro/src/runtime/client/load.ts`
- `packages/astro/src/runtime/client/idle.ts`
- `packages/astro/src/runtime/client/visible.ts`
- `packages/astro/src/runtime/client/media.ts`
- `packages/astro/src/runtime/client/only.ts`

These directive implementations are intentionally thin. They mostly do three things:

1. wait for a trigger condition
2. call `load()` to fetch the hydrator
3. execute hydration

Their meaning is:

- `client:load`: hydrate as soon as the page is ready
- `client:idle`: hydrate during browser idle time
- `client:visible`: hydrate when the island becomes visible
- `client:media`: hydrate when a media query matches
- `client:only`: skip SSR and render only on the client

Architecturally, Astro's client directives are not a second rendering system. They are just scheduling hooks for island activation.

---

## 8. Server islands: `server:defer`

Astro also supports server islands in addition to client islands.

Key files:

- `packages/astro/src/runtime/server/render/server-islands.ts`
- `packages/astro/src/core/server-islands/endpoint.ts`
- `packages/astro/src/core/server-islands/vite-plugin-server-islands.ts`

### Identifying a server island

In:

- `packages/astro/src/runtime/server/render/component.ts`

components with `server:*` metadata are routed into `ServerIslandComponent`.

### Rendering fallback first, then replacing it

`ServerIslandComponent` does not immediately inline the final HTML into the page. Instead it:

1. renders fallback content
2. emits a runtime script
3. requests `/_server-islands/[name]`
4. replaces the placeholder with the fetched HTML

### Encrypting the payload

In:

- `packages/astro/src/runtime/server/render/server-islands.ts`

the component export name, props, and slots are encrypted before being sent through GET or POST.

That protects internal component metadata and reduces the risk of request tampering.

### Returning HTML from the endpoint

In:

- `packages/astro/src/core/server-islands/endpoint.ts`

Astro:

1. parses the request
2. decrypts the export name, props, and slots
3. loads the right component module
4. renders it on the server
5. returns an HTML fragment

This differs from client islands. Client islands exist to hydrate interactive UI in place; server islands exist to defer a server render and fill the result back into the page later.

---

## 9. Build time: collecting the server island manifest

Key file:

- `packages/astro/src/core/server-islands/vite-plugin-server-islands.ts`

This Vite plugin:

- discovers server islands from `meta.astro.serverComponents`
- generates names and module-path mappings for them
- builds `virtual:astro:server-island-manifest`

At runtime, that manifest lets Astro:

- map a server island name back to a real component module
- supply the import map used by the `/_server-islands/[name]` endpoint

---

## 10. The most important files to read

If you only want the core implementation, start with these files.

### Client islands

1. `packages/astro/src/vite-plugin-astro/index.ts`
   - compiler output for `hydratedComponents`, `clientOnlyComponents`, and `serverComponents`

2. `packages/astro/src/runtime/server/hydration.ts`
   - parses `client:*` directives
   - generates hydration metadata for `<astro-island>`

3. `packages/astro/src/runtime/server/render/component.ts`
   - decides whether a component becomes an island
   - emits `<astro-island>`

4. `packages/astro/src/runtime/server/scripts.ts`
   - injects directive scripts and the island runtime

5. `packages/astro/src/runtime/server/astro-island.ts`
   - the core browser runtime for islands

6. `packages/astro/src/runtime/client/load.ts`
7. `packages/astro/src/runtime/client/idle.ts`
8. `packages/astro/src/runtime/client/visible.ts`
9. `packages/astro/src/runtime/client/media.ts`
10. `packages/astro/src/runtime/client/only.ts`
    - define when each built-in `client:*` directive triggers hydration

### Server islands

11. `packages/astro/src/runtime/server/render/server-islands.ts`
    - creates the `server:defer` placeholder and replacement script

12. `packages/astro/src/core/server-islands/endpoint.ts`
    - returns the HTML for a deferred server island

13. `packages/astro/src/core/server-islands/vite-plugin-server-islands.ts`
    - collects server islands and builds the manifest

---

## 11. Summary

Astro's island architecture is fundamentally:

1. **compile-time detection of components marked with `client:*` or `server:*`**
2. **server-rendered static HTML as the initial output**
3. **wrapping interactive regions in `<astro-island>`**
4. **serializing component URLs, renderer URLs, props, and hydration strategy onto that element**
5. **hydrating those regions later through the `astro-island` custom element**
6. **using `server:defer` to fetch deferred server-rendered HTML from a dedicated endpoint**

So Astro islands are not a separate renderer. They are a cross-cutting mechanism spanning:

- compiler metadata
- SSR output
- page script injection
- browser runtime hydration
- deferred server rendering
