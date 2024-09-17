# Préchargements

L'objectif de cette page est de faire un état des lieux du comportement natif de Vite et des choix qui ont été faits pour copier son comportement dans une application Symfony.
La lecture de cette documentation peut vous être nécessaire si vous cherchez à paramétrer finement le préchargement des scripts et feuilles de style ou si vous souhaitez contribuer au projet. Elle traitera de sujets comme les attributs `rel="preload"` et `rel="modulepreload"` et `crossorigin="xxx"`.

## TL;DR;

code source utilisé pour la documentation de cette page

```js
// assets/app.js
import { foo } from "./shared.js";
console.log(foo);

window.addEventListener("DOMContentLoaded", () => {
  import("./my-async-dep.ts").then(({ hello }) => {
    console.log(hello);
  });
});
```

Comportement natif de Vite : analyse du code source du fichier `index.html` en dev mode

```html
<html lang="en">
  <head>
    <script type="module" src="/@vite/client"></script>
    <script type="module" src="/assets/app.js"></script>
  </head>
  <body>
    ...
  </body>
</html>
```

Comportement souhaité

Afin d'être exhaustif et parce que l'application Symfony est servie sur une autre URL. l'URL complète
du client Vite devra être écrite. On ajoute explicitement l'attribut `crossorigin`.
```html
<html lang="en">
  <head>
    <script type="module" src="http://127.0.0.1:5173/@vite/client" crossorigin></script>
    <script type="module" src="http://127.0.0.1:5173/assets/app.js" crossorigin></script>
  </head>
  <body>
    ...
  </body>
</html>
```


Comportement natif de Vite : analyse du code source du fichier `index.html` après un build

```html
<html lang="en">
  <head>
    <!-- l'attribut crossorigin est présent par défaut dans toutes les balises script et link -->
    <script type="module" crossorigin src="/assets/app-bXh9WZ8a.js"></script>
    <!-- les scripts qui ont été splittés sont préchargés -->
    <link rel="modulepreload" crossorigin href="/assets/shared-5Hh7diCN.js">
    <link rel="stylesheet" crossorigin href="/assets/index-Cz4zGhbH.css">
  </head>
  <body>
    ...
  </body>
</html>
```

Comportement souhaité avec un préchargement sous forme de balises html :

Exactement le même rendu que pour le comportement natif

Comportement souhaité avec un préchargement sous forme d'en-tête `Link` :

bien que explicitement présent dans le code source du fichier html le fichier `/assets/app-bXh9WZ8a.js`
sera tout de même présent dans l'en-tête `Link`. Si correctement configuré cela ne provoque pas de
requête supplémentaire mais un requête exécutée encore plus tôt.

```bash
Link: </assets/app-bXh9WZ8a.js>; rel="modulepreload"; crossorigin, </assets/shared-5Hh7diCN.js>; rel="modulepreload"; crossorigin
```
```html
<!doctype html>
<html lang="en">
  <head>
    <!-- l'attribut crossorigin est présent par défaut dans toutes les balises script et link -->
    <script type="module" crossorigin src="/assets/app-bXh9WZ8a.js"></script>
    <link rel="stylesheet" crossorigin href="/assets/index-Cz4zGhbH.css">
  </head>
  <body>
    ...
  </body>
</html>
```

Comportement natif de Vite : analyse du code source du fichier `index.html` après un build avec le plugin legacy

```html
<!doctype html>
<html class="page-welcome">
  <head>
    <meta charset="UTF-8" />
    <title>HelloController!</title>
    <link rel="icon" href="data:;base64,iBORw0KGgo=" />

    <link rel="modulepreload" crossorigin href="/assets/shared-5Hh7diCN.js">
    <link rel="stylesheet" crossorigin href="/build/assets/theme-gOv5yEUY.css" />
    <script type="module" crossorigin src="/build/assets/pageWelcome--4K79RzD.js"></script>

    <script type="module">/** DETECT_MODERN_BROWSER_INLINE_CODE */</script>
    <script type="module">/** DYNAMIC_FALLBACK_INLINE_CODE */</script>
  </head>

  <body>
    <script nomodule>/** SAFARI10_NO_MODULE_FIX_INLINE_CODE */</script>
    <script nomodule crossorigin src="/build/assets/polyfills-legacy-im0EAdDu.js" id="vite-legacy-polyfill" ></script>
    <script
      nomodule
      data-src="/build/assets/pageWelcome-legacy-TXGWt7m6.js"
      id="vite-legacy-entry-page-welcome"
      crossorigin
      class="vite-legacy-entry"
    >
      System.import(
        document.getElementById('vite-legacy-entry-page-welcome').getAttribute('data-src')
      );
    </script>
  </body>
</html>
```

## Attribut `crossorigin` activé par défaut dans la version 7 de `vite-bundle`.

Pourquoi ?

- On s'approche au plus près du comportement natif de Vite.
- Normalement le mode [`cors`](https://developer.mozilla.org/en-US/docs/Web/API/Request/mode) est activé implicitement pour toute balise `<script type="module">`. En ajoutant explicitement l'attribut `crossorigin` cela ne doit rien changer. Mais cela permet aux polyfills comme celui de [`modulePreload`](https://vitejs.dev/config/build-options.html#build-modulepreload) de mieux fonctionner. En effet sans la présence de cet attribut, le polyfill pourrait effectuer un fetch inutile

```
A preload for 'https://example.org/build/example.js' is found, but is not used because the request credentials mode does not match. Consider taking a look at crossorigin attribute.
```


```ts
// from vite source code
// https://github.com/vitejs/vite/blob/1bda847329022d5279cfa2b51719dd19a161fd64/packages/vite/src/node/plugins/html.ts#L713
const toScriptTag = (
  chunk: OutputChunk,
  toOutputPath: (filename: string) => string,
  isAsync: boolean,
): HtmlTagDescriptor => ({
  tag: 'script',
  attrs: {
    ...(isAsync ? { async: true } : {}),
    type: 'module',
    // crossorigin must be set not only for serving assets in a different origin
    // but also to make it possible to preload the script using `<link rel="preload">`.
    // `<script type="module">` used to fetch the script with credential mode `omit`,
    // however `crossorigin` attribute cannot specify that value.
    // https://developer.chrome.com/blog/modulepreload/#ok-so-why-doesnt-link-relpreload-work-for-modules:~:text=For%20%3Cscript%3E,of%20other%20modules.
    // Now `<script type="module">` uses `same origin`: https://github.com/whatwg/html/pull/3656#:~:text=Module%20scripts%20are%20always%20fetched%20with%20credentials%20mode%20%22same%2Dorigin%22%20by%20default%20and%20can%20no%20longer%0Ause%20%22omit%22
    crossorigin: true,
    src: toOutputPath(chunk.fileName),
  },
})
```

l'option `build.modulePreload.polyfill` injecte ce code supplémentaire qui va créer explicitement un fetch sur les balises `<link rel="modulepreload" href="/script.js />` trouvées dans le code source.

- le polyfill fonctionne uniquement pour `modulepreload` pas pour `preload`
- grâce au `MutationObserver` le polyfill préchargera les `modulepreload` créés tout au long du cycle de vie de la page.
- le polyfill est présent dans le fichier js du point d'entrée il n'est pas injecté dans le fichier `index.html`,
 pas besoin de l'ajouter nous-même.

le code source est inspiré de [es-module-shims](https://github.com/guybedford/es-module-shims/blob/13694283ec3b6aafdfd91ca1033df9f5f34bd4cf/src/es-module-shims.js#L603).

```ts
// source code from vite
// https://github.com/vitejs/vite/blob/1bda847329022d5279cfa2b51719dd19a161fd64/packages/vite/src/node/plugins/modulePreloadPolyfill.ts#L59
(function polyfill() {
  const relList = document.createElement('link').relList
  if (relList && relList.supports && relList.supports('modulepreload')) {
    return
  }

  for (const link of document.querySelectorAll('link[rel="modulepreload"]')) {
    processPreload(link)
  }

  new MutationObserver((mutations: any) => {
    for (const mutation of mutations) {
      if (mutation.type !== 'childList') {
        continue
      }
      for (const node of mutation.addedNodes) {
        if (node.tagName === 'LINK' && node.rel === 'modulepreload')
          processPreload(node)
      }
    }
  }).observe(document, { childList: true, subtree: true })

  function getFetchOpts(link: any) {
    const fetchOpts = {} as any
    if (link.integrity) fetchOpts.integrity = link.integrity
    if (link.referrerPolicy) fetchOpts.referrerPolicy = link.referrerPolicy
    if (link.crossOrigin === 'use-credentials')
      fetchOpts.credentials = 'include'
    else if (link.crossOrigin === 'anonymous') fetchOpts.credentials = 'omit'
    else fetchOpts.credentials = 'same-origin'
    return fetchOpts
  }

  function processPreload(link: any) {
    if (link.ep)
      // ep marker = processed
      return
    link.ep = true
    // prepopulate the load record
    const fetchOpts = getFetchOpts(link)
    fetch(link.href, fetchOpts)
  }
})();
```

Quelque chose semble étonnant `if (link.crossOrigin === 'anonymous') fetchOpts.credentials = 'omit'`.
Je n'avais jamais vu cela [crossorigin](https://developer.mozilla.org/en-US/docs/Web/API/HTMLScriptElement/crossOrigin). dans mon esprit `anonymous` est associé la valeur `same-origin` pour `credentials`.


# Annexes



## Fichiers générés lors d'un build sans le plugin Legacy



```json
// entrypoints.json
{
  "base": "/build/",
  "entryPoints": {
    "app": {
      "css": [
        "/build/assets/app-Cz4zGhbH.css"
      ],
      "dynamic": [
        "/build/assets/my-async-dep-BSHE5Y3H.js"
      ],
      "js": [
        "/build/assets/app-BBraN-Dm.js"
      ],
      "legacy": false,
      "preload": [
        "/build/assets/shared-5Hh7diCN.js"
      ]
    }
  },
  "legacy": false,
  "metadatas": {},
  "version": ["6.5.0", 6, 5, 0],
  "viteServer": null
}

// manifest.json
{
  "_shared-5Hh7diCN.js": {
    "file": "assets/shared-5Hh7diCN.js",
    "name": "shared"
  },
  "index.html": {
    "file": "assets/app-bXh9WZ8a.js",
    "name": "index",
    "src": "index.html",
    "isEntry": true,
    "imports": [
      /* fichiers qui doivent être préchargés */
      "_shared-5Hh7diCN.js"
    ],
    "dynamicImports": [
      /* fichiers qui ne devraient pas être préchargés */
      "src/my-async-dep.ts"
    ],
    "css": [
      "assets/index-Cz4zGhbH.css"
    ],
    /* les assets ne sont pas préchargés */
    "assets": [
      "assets/typescript-EnIy2PE5.svg"
    ]
  },
  "src/my-async-dep.ts": {
    "file": "assets/my-async-dep-BSHE5Y3H.js",
    "name": "my-async-dep",
    "src": "src/my-async-dep.ts",
    "isDynamicEntry": true
  },
  "src/typescript.svg": {
    "file": "assets/typescript-EnIy2PE5.svg",
    "src": "src/typescript.svg"
  }
}
```



```js
// assets/app-bXh9WZ8a.js

// on retrouve nos fichiers qui doivent être préchargés qui sont directement
// importés dans le code js.
import { foo } from "./shared-5Hh7diCN.js";
console.log(foo);

// nos imports dynamiques sont enveloppés dans une fonction `preloadResources`
// qui ne fait rien de particulier.
window.addEventListener("DOMContentLoaded", () => {
  preloadResources(async () => {
    const { hello } = await import("./my-async-dep-BSHE5Y3H.js");
    return { hello };
  }, []).then(({ hello }) => {
    console.log(hello);
  });
});

```




## Fichiers générés avec le plugin Legacy


```json
// entrypoints.json
{
  "base": "/build/",
  "entryPoints": {
    "pageWelcome-legacy": {
      "css": [],
      "dynamic": [],
      "js": [
        "/build/assets/pageWelcome-legacy-TXGWt7m6.js"
      ],
      "legacy": false,
      "preload": [
        "/build/assets/shared-legacy-Gun4aQJC.js"
      ]
    },
    "pageWelcome": {
      "css": [],
      "dynamic": [],
      "js": [
        "/build/assets/pageWelcome--4K79RzD.js"
      ],
      "legacy": "pageWelcome-legacy",
      "preload": [
        "/build/assets/shared-5Hh7diCN.js"
      ]
    },
    "theme-legacy": {
      "css": [],
      "dynamic": [],
      "js": [
        "/build/assets/theme-legacy-LruLDnf0.js"
      ],
      "legacy": false,
      "preload": []
    },
    "theme": {
      "css": [
        "/build/assets/theme-gOv5yEUY.css"
      ],
      "dynamic": [],
      "js": [],
      "legacy": "theme-legacy",
      "preload": []
    },
    "polyfills-legacy": {
      "css": [],
      "dynamic": [],
      "js": [
        "/build/assets/polyfills-legacy-im0EAdDu.js"
      ],
      "legacy": false,
      "preload": []
    }
  },
  "legacy": true,
  "metadatas": {},
  "version": ["6.5.0", 6, 5, 0],
  "viteServer": null
}
```
```json
// manifest.json
{
  "_shared-5Hh7diCN.js": {
    "file": "assets/shared-5Hh7diCN.js",
    "name": "shared"
  },
  "_shared-legacy-Gun4aQJC.js": {
    "file": "assets/shared-legacy-Gun4aQJC.js",
    "name": "shared"
  },
  "assets/page/welcome/index-legacy.js": {
    // legacy browsers only load this script
    "file": "assets/pageWelcome-legacy-TXGWt7m6.js",
    "isEntry": true,
    "src": "assets/page/welcome/index-legacy.js"
  },
  "assets/page/welcome/index.js": {
    // modern browsers only load this script
    "file": "assets/pageWelcome--4K79RzD.js",
    "isEntry": true,
    "src": "assets/page/welcome/index.js"
  },
  "assets/theme-legacy.scss": {
    "assets": [
      "assets/topography-iIXb0l18.svg"
    ],
    "file": "assets/theme-legacy-LruLDnf0.js",
    "isEntry": true,
    "src": "assets/theme-legacy.scss"
  },
  "assets/theme.scss": {
    "file": "assets/theme-gOv5yEUY.css",
    "isEntry": true,
    "src": "assets/theme.scss"
  },
  "vite/legacy-polyfills-legacy": {
    // polyfill loaded only once
    "file": "assets/polyfills-legacy-im0EAdDu.js",
    "isEntry": true,
    "src": "vite/legacy-polyfills-legacy"
  }
}
```

```html
<!doctype html>
<html class="page-welcome">
  <head>
    <meta charset="UTF-8" />
    <title>HelloController!</title>
    <link rel="icon" href="data:;base64,iBORw0KGgo=" />

    <!-- seule la version moderne de shared-XXX.js est préchargée -->
    <!-- les balises link et script des fichiers non-legacy ont les mêmes attributs que le plugin legacy -->
    <!-- soit actif ou pas -->
    <link rel="modulepreload" crossorigin href="/assets/shared-5Hh7diCN.js">
    <link rel="stylesheet" crossorigin href="/build/assets/theme-gOv5yEUY.css" />
    <script type="module" crossorigin src="/build/assets/pageWelcome--4K79RzD.js"></script>

    <script type="module">
      // DETECT_MODERN_BROWSER_INLINE_CODE
      // le code a changé.
      // readable version with chatGPT
      try {
          // Check if the current environment supports the "import.meta" syntax
          const metaUrl = import.meta.url;
          // Attempt to import the "_" module and catch any errors
          import("_").catch(() => {});
      } catch (e) {
          // Do nothing if the "import" syntax or "import.meta" is not supported
      }

      // Set a global variable indicating that the browser is modern
      window.__vite_is_modern_browser = true;
    </script>
    <script type="module">
      // DYNAMIC_FALLBACK_INLINE_CODE
      // readable version with chatGPT
      (function() {
        if (window.__vite_is_modern_browser) {
            return;
        }

        console.warn("vite: loading legacy build because dynamic import or import.meta.url is unsupported, syntax error above should be ignored");

        var legacyPolyfill = document.getElementById("vite-legacy-polyfill");
        var script = document.createElement("script");
        script.src = legacyPolyfill.src;
        script.onload = function() {
            System.import(document.getElementById('vite-legacy-entry').getAttribute('data-src'))
        };
        document.body.appendChild(script);
      })();
    </script>
  </head>

  <body>
    <!-- ... -->
    <script nomodule>
      // SAFARI10_NO_MODULE_FIX_INLINE_CODE
      // readable version with chatGPT
      (function() {
          var document = window.document;
          var script = document.createElement("script");

          // Check if the "nomodule" attribute is supported and the "onbeforeload" event is supported
          if (!("noModule" in script) && "onbeforeload" in script) {
              var nomoduleFound = false;

              // Add a "beforeload" event listener
              document.addEventListener("beforeload", function(event) {
                  if (event.target === script) {
                      nomoduleFound = true;
                  } else if (!event.target.hasAttribute("nomodule") || !nomoduleFound) {
                      return;
                  }
                  event.preventDefault();
              }, true);

              script.type = "module";
              script.src = ".";
              document.head.appendChild(script);
              script.remove();
          }
      })();
    </script>
    <!-- les balises script pour le plugin -->
    <!-- 1/ n'ont pas l'attribut type="module" mais l'attribut nomodule à la place-->
    <!-- 2/ ont l'attribut crossorigin comme pour les autres balises -->
    <script
      nomodule
      crossorigin
      src="/build/assets/polyfills-legacy-im0EAdDu.js"
      id="vite-legacy-polyfill"
    ></script>
    <!-- pour les points d'entrée qui sont css only, le fichier **-XXX.js associé ne devrait pas être nécessaire. -->
    <script
      nomodule
      data-src="/build/assets/theme-legacy-LruLDnf0.js"
      id="vite-legacy-entry-theme"
      crossorigin
      class="vite-legacy-entry"
    >
      System.import(document.getElementById('vite-legacy-entry-theme').getAttribute('data-src'));
    </script>

    <script
      nomodule
      data-src="/build/assets/pageWelcome-legacy-TXGWt7m6.js"
      id="vite-legacy-entry-page-welcome"
      crossorigin
      class="vite-legacy-entry"
    >
      System.import(
        document.getElementById('vite-legacy-entry-page-welcome').getAttribute('data-src')
      );
    </script>
  </body>
</html>

```
