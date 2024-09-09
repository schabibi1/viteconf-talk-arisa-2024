---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: ViteConf talk script
info: |
  ## ViteConf talk 2024 slides, Arisa Fukuzaki, Enhancing DX with Vite open source libraries
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# Enhancing DX with Vite open source libraries

<div class="pt-12">Arisa Fukuzaki</div>
Senior DevRel Engineer at Storyblok

<!--
Hi everyone👋 My name is Arisa. I’m a senior DevRel Engineer at Storyblok.

Hope you’re enjoying the ViteConf.

This talk is about “Enhancing DX with Vite open source libraries” to show you a real-world library implemented with Vite, used by thousands of users.

You will see how Vite streamlines integration and optimizes the development process.

It also makes it easier for developers to build open source packages efficiently.
-->

---

# Astro Integration API

i.e. `astro:config:setup`
<div>
  <p>Astro Integration API has features to add new functionality and behaviors for your project to write your own integration.</p>
  <p>It provides built-in hooks that integrations can implement to execute during certain parts of Astro’s lifecycle.</p>
</div>

```ts {*|5|8-9|11|*}
'astro:config:setup'?: (options: {
  config: AstroConfig;
  command: 'dev' | 'build' | 'preview' | 'sync';
  isRestart: boolean;
  updateConfig: (newConfig: DeepPartial<AstroConfig>) => AstroConfig;
  addRenderer: (renderer: AstroRenderer) => void;
  addClientDirective: (directive: ClientDirectiveConfig) => void;
  addMiddleware: (middleware: AstroIntegrationMiddleware) => void;
  addDevToolbarApp: (pluginEntrypoint: string) => void;
  addWatchFile: (path: URL | string) => void;
  injectScript: (stage: InjectedScriptStage, content: string) => void;
  injectRoute: ({ pattern: string, entrypoint: string }) => void;
  logger: AstroIntegrationLogger;
}) => void | Promise<void>;
```

[Learn more](https://docs.astro.build/en/reference/integrations-reference/#astroconfigsetup)

<!--
We cover two important features, Astro Integration API and Vite Virtual Modules.

Astro Integration API has features to add new functionality and behaviors for your project to write your own integration.

Astro Integration API provides built-in integration hooks to implement and execute during certain parts of Astro’s lifecycle.

In our @storyblok/astro SDK, we use `astro:config:setup` for initialization to extend the project config.

We use `updateConfig` (highlight), `addMiddleware`, `addDevToolbarApp` (highlight), and `injectScript` (highlight) options.
-->

---

# Vite Virtual Modules
<div>
  <p>Virtual modules are a useful scheme that allows you to pass build time information to the source files using normal ESM import syntax. Which allows importing the module in JavaScript:</p>
</div>

````md magic-move
```js
export default function myPlugin() {
  const virtualModuleId = 'virtual:my-module'
  const resolvedVirtualModuleId = '\0' + virtualModuleId // plugins that use virtual modules prefix module ID with \0

  return {
    name: 'my-plugin', // required, will show up in warnings and errors
    resolveId(id) {
      if (id === virtualModuleId) {
        return resolvedVirtualModuleId
      }
    },
    load(id) {
      if (id === resolvedVirtualModuleId) {
        return `export const msg = "from virtual module"`
      }
    },
  }
}
```

```js
import { msg } from 'virtual:my-module'

console.log(msg)
```
````
[Learn more](https://vitejs.dev/guide/api-plugin#virtual-modules-convention)

<!-- Virtual modules are a useful scheme that allows you to pass build time information to the source files using normal ESM import syntax.

(Magic move) This means we can import the modules into JavaScript like this. -->

---

# `vite-plugin-storyblok-init.ts`

```ts {*|6-7|*}
export function vitePluginStoryblokInit(
  accessToken: string,
  useCustomApi: boolean,
  apiOptions: ISbConfig
): Plugin {
  const virtualModuleId = "virtual:storyblok-init";
  const resolvedVirtualModuleId = "\0" + virtualModuleId; // plugins that use virtual modules prefix module ID with \0
  return {
    name: "vite-plugin-storyblok-init",// required
    async resolveId(id: string) { if (id === virtualModuleId) resolvedVirtualModuleId },
    async load(id: string) {
      if (id === resolvedVirtualModuleId) {
        return `
          import { storyblokInit, apiPlugin } from "@storyblok/js";
          const { storyblokApi } = storyblokInit({
            accessToken: "${accessToken}",
            use: ${useCustomApi ? "[]" : "[apiPlugin]"},
            apiOptions: ${JSON.stringify(apiOptions)},
          });
          export const storyblokApiInstance = storyblokApi;
        `;
      }
    },
  };
}
```

<!--
Let's take a look at the `vite-plugin-storyblok-init.ts` file.

We use the Virtual Modules to create a virtual module that sets up the Storyblok API instance here.

(Highlight) Virtual modules in Vite are prefixed with `virtual:` for the user-facing path by convention.

This means that `vite-plugin-storyblok-init` asks users to import a `virtual: storyblok-init` virtual module to get build time information.

Plugins that use virtual modules need to prefix module ID with `\0`.
-->

---

# `lib/index.ts`

````md magic-move
```ts {*|15-16|*}
import { vitePluginStoryblokInit } from "./vite-plugins/vite-plugin-storyblok-init";
import type { AstroGlobal, AstroIntegration } from "astro";
// ...
export type IntegrationOptions = {
  accessToken: string;
  useCustomApi?: boolean;
  apiOptions?: ISbConfig; // ...
};
// ...
export default function storyblokIntegration(
  options: IntegrationOptions
): AstroIntegration {
  const resolvedOptions = { ...options, /* Storyblok-related options */};
  return { // ...
    hooks: {
      "astro:config:setup": ({/* astro:config:setup hook options */}) => {
        updateConfig({
          vite: {
            plugins: [
              vitePluginStoryblokInit(
                resolvedOptions.accessToken,
                resolvedOptions.useCustomApi,
                resolvedOptions.apiOptions
              ), // ...

            ],
          },
        });
      },
    },
  };
}
```

```ts {*|9-12|*}
// ...
export default function storyblokIntegration(
  options: IntegrationOptions
): AstroIntegration {
  const resolvedOptions = { ...options, /* Storyblok-related options */};
  return { // ...
    hooks: {
      "astro:config:setup": ({
        injectScript,
        updateConfig,
        addDevToolbarApp,
        addMiddleware,
        config,
      }) => {
        updateConfig({
          vite: {
            plugins: [
              vitePluginStoryblokInit(
                resolvedOptions.accessToken,
                resolvedOptions.useCustomApi,
                resolvedOptions.apiOptions
              ), // ...
            ],
          }, // ...

        });
      },
    },
  };
}
```

```ts {*|18-19|*}
updateConfig({
  vite: {
    plugins: [
      vitePluginStoryblokInit(
        resolvedOptions.accessToken,
        resolvedOptions.useCustomApi,
        resolvedOptions.apiOptions
      ), // ...
    ],
  },
});
if (resolvedOptions.livePreview && config?.output !== "server") {
  throw new Error(/* Astro Storyblok Live feature error message to disable SSR */);
}
injectScript(
  "page-ssr",
  `
  import { storyblokApiInstance } from "virtual:storyblok-init";
  globalThis.storyblokApiInstance = storyblokApiInstance;
  `
);
```

```ts
// ...
export function useStoryblokApi(): StoryblokClient {
  if (!globalThis.storyblokApiInstance) {
    console.error("storyblokApiInstance has not been initialized correctly");
  }
  return globalThis.storyblokApiInstance;
}
// ...
```

```ts {*|4-9|*}
import { vitePluginStoryblokInit } from "./vite-plugins/vite-plugin-storyblok-init";
import type { AstroGlobal, AstroIntegration } from "astro";
// ...
export type IntegrationOptions = {
  accessToken: string;
  useCustomApi?: boolean;
  apiOptions?: ISbConfig;
   // ...
};
// ...
export default function storyblokIntegration(
  options: IntegrationOptions
): AstroIntegration {/* ... */}
```

```ts
updateConfig({
  vite: {
    plugins: [
      vitePluginStoryblokInit(
        resolvedOptions.accessToken,
        resolvedOptions.useCustomApi,
        resolvedOptions.apiOptions
      ), // ...
    ],
  },
});
if (resolvedOptions.livePreview && config?.output !== "server") {
  throw new Error(/* Astro Storyblok Live feature error message to disable SSR */);
}
injectScript(
  "page-ssr",
  `
  import { storyblokApiInstance } from "virtual:storyblok-init";
  globalThis.storyblokApiInstance = storyblokApiInstance;
  `
);
```
````

[Learn more, `globalThis`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/globalThis)

[Learn more, `injectScrupt`](https://docs.astro.build/en/reference/integrations-reference/#injectscript-option)

<!--
(Highlight) The default function is from the Astro integration API side of the feature such as `astro:config:setup` hooks.

(Magic move & highlight) We use `injectScript`, `updateConfig`, `addDevToolbarApp`, and `addMiddleware` as we saw together with the Astro Integration API.

(Magic move & highlight) Here, we import `storyblokApiInstance` from the virtual module file.

Also, we use `globalThis` variable to create a new property of the object assigned in the Storyblok API instance, which we import from the virtual module file.

(Magic move) After that, we use it in the `useStoryblokApi` function, which we use to export this function to be used in Astro files.

`injectScript` imports `storyblokApiInstance` from the virtual module file.

It looks like we’re making a detour, but there’s a reason.

(Magic move) We want to achieve to initialize `storyblokApiInstance` directly via the Astro Integration API to improve DX.

(Highlights) The access token and API options are passed to the virtual module via the `updateConfig` function of the `astro:config:setup`.

`updateConfig` lets you update the Vite configuration directly from the Astro Integration API.

`vitePluginStoryblokInit` uses these parameters to create `storyblokApiInstance` using `storyblokInit` from `@storyblok/js` SDK.

This is exposed as a virtual file.

It can be used to import `storyblokApiInstance` in the Astro runtime.

(Magic move) This process happens via the `injectScript`.

`storyblokApiInstance` is created as a property of `globalThis` to allow `useStoryblokApi` function to work and make runtime information available in the build.
-->

---

# `vite-plugin-storyblok-components.ts` & `index.ts`

````md magic-move
```ts {*|1-6|*}
export function vitePluginStoryblokComponents(
  componentsDir: string,
  components?: object,
  enableFallbackComponent?: boolean,
  customFallbackComponent?: string
): Plugin {
  const virtualModuleId = "virtual:storyblok-components";
  const resolvedVirtualModuleId = "\0" + virtualModuleId;
  return {
    name: "vite-plugin-storyblok-components",
    async resolveId(id: string) {
      if (id === virtualModuleId) {
        return resolvedVirtualModuleId;
      }
    },
    async load(id: string) {
      if (id === resolvedVirtualModuleId) {
        // Handle registered components, custom fallback component, etc
      }
    },
  };
}
```

```ts {*|2-7|17-22|*}
import { vitePluginStoryblokComponents } from "./vite-plugins/vite-plugin-storyblok-components";
export type IntegrationOptions = {
  componentsDir?: string;
  components?: object;
  customFallbackComponent?: string;
  enableFallbackComponent?: boolean; // ...
};
// ...
export default function storyblokIntegration(options: IntegrationOptions): AstroIntegration {
  const resolvedOptions = { ...options, /* Storyblok-related options */};
  return { // ...
    hooks: {
      "astro:config:setup": ({/* astro:config:setup hook options */}) => {
        updateConfig({
          vite: {
            plugins: [
              vitePluginStoryblokComponents(
                resolvedOptions.componentsDir,
                resolvedOptions.components,
                resolvedOptions.enableFallbackComponent,
                resolvedOptions.customFallbackComponent
              ), // ...

            ],
          },
        });
      },
    },
  };
}
```
````

<!--
(Highlight) We pass parameters like `componentsDir`, `components`, `enableFallbackComponent`, and `customFallbackComponent` to set up a custom components directory where actual components, fallback components, and custom fallback components are registered.

(Magic move & highlight) We see the same pattern as `vitePluginStoryblokInit` (highlight) where these parameters are defined and where they are passed to the module.
-->

---

# Thank you!

[@storyblok-astro SDK repo](https://github.com/storyblok/storyblok-astro) · [Virtual Modules docs](https://vitejs.dev/guide/api-plugin#virtual-modules-convention) · [Astro Integration API docs](https://docs.astro.build/en/reference/integrations-reference/) · [MDN, `globalThis`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/globalThis)

<PoweredBySlidev mt-10 />

<!--
Thanks for watching my talk🙏 
Here are all the resources in my talk, including our @storyblok/astro SDK repository.

Our JS ecosystem SDKs are open-source, and please feel free to have a look at them.
You can contribute if it also helps other users using our SDKs.

And huge thanks to my co-maintainers and all the contributors who helped us keep our Storyblok Astro SDK in good shape!

Let’s enjoy the rest of the ViteConf talks! See you soon👋
-->
