> Reproduction for https://github.com/vitejs/vite-plugin-react-swc/issues/74

## TLDR

I think the [`Bindings not found` error](https://github.com/vitejs/vite-plugin-react-swc/issues/74) occurs when the lockfile is missing an entry for `@swc/core-xxxx` platform-specific binary for the current platform.

## Problem

In a Vite + React + SWC project, this error can occur when starting Vite's dev server especially after refreshing the `package-lock.json` file without deleting `node_modules` beforehand:

```sh
Bindings not found.
12:08:10 AM [vite] Internal server error: Bindings not found.
  Plugin: vite:react-swc
  File: /Users/tony/src/tmp/vite-plugin-react-swc-bug-demo/src/main.tsx
      at Compiler.<anonymous> (/Users/tony/src/tmp/vite-plugin-react-swc-bug-demo/node_modules/@swc/core/index.js:225:19)
      at Generator.next (<anonymous>)
      at /Users/tony/src/tmp/vite-plugin-react-swc-bug-demo/node_modules/@swc/core/index.js:34:71
      at new Promise (<anonymous>)
      at __awaiter (/Users/tony/src/tmp/vite-plugin-react-swc-bug-demo/node_modules/@swc/core/index.js:30:12)
      at Compiler.transform (/Users/tony/src/tmp/vite-plugin-react-swc-bug-demo/node_modules/@swc/core/index.js:202:16)
      at transform (/Users/tony/src/tmp/vite-plugin-react-swc-bug-demo/node_modules/@swc/core/index.js:344:21)
      at transformWithOptions (file:///Users/tony/src/tmp/vite-plugin-react-swc-bug-demo/node_modules/@vitejs/plugin-react-swc/index.mjs:130:20)
      at TransformContext.transform (file:///Users/tony/src/tmp/vite-plugin-react-swc-bug-demo/node_modules/@vitejs/plugin-react-swc/index.mjs:55:30)
      at Object.transform (file:///Users/tony/src/tmp/vite-plugin-react-swc-bug-demo/node_modules/vite/dist/node/chunks/dep-a178814b.js:42877:44)
```

This might be happening because `@swc/core`'s post-install script (which presumably installs the `@swc/core-xxxx` binary for the current platform) doesn't run in the scenario above.

## Steps to reproduce

1. Scaffold a new Vite project with React+SWC (or clone this repo), and CD into the project directory:

    ```sh
    npm init vite@latest vite-plugin-react-swc-bug-demo -- --template react-swc
    cd vite-plugin-react-swc-bug-demo
    ```

2. Install dependencies:

    ```sh
    npm install
    ```

3. Observe the resulting lockfile contains `node_modules/@swc/core-xxxx` entries for all supported platforms:

    ```sh
    grep -E 'node_modules/@swc/core-.*' package-lock.json
    ```

3. Refresh the lockfile by deleting it and reinstalling dependencies (NOTE: Do not delete `node_modules` beforehand):

    ```sh
    rm package-lock.json
    npm install
    ```

5. Observe the resulting lockfile contains only the `@swc/core-xxxx` binary for the current platform:

    ```sh
    grep -E 'node_modules/@swc/core-.*' package-lock.json
    ```

6. Observe the corresponding `node_modules/@swc/core-xxxx` exists:

    ```sh
    ls ./node_modules/@swc/core-*
    ```

6. To simulate what occurs when an `npm install` is run on a different platform, delete the `@swc/core-xxxx` entry in `package-lock.json` and regenerate the lockfile.

    ```sh
    # Manually delete the @swc/core-xxxx entry from package-lock.json ...
    # OR run this jq command (edit platform as necessary):
    jq -r 'del(.packages."node_modules/@swc/core-darwin-arm64")' package-lock.json > package-lock.json.new && mv package-lock.json.new package-lock.json

    # and then regenerate package-lock.json
    npm install
    ```

7. Observe the module from step 6 no longer exists:

    ```sh
    ls ./node_modules/@swc/core-*
    ```

8. Start Vite's dev server:

    ```sh
    npm run dev
    ```

9. Open http://localhost:5173 (or the port that Vite chooses)

10. See the `Bindings not found` error from `vite:react-swc` in the terminal


## Workaround

The most reliable workaround I found is to force-install the optional dependencies of `@swc/core` as optional dependencies of your own project:

```shell
npm install --force --save-optional \
        @swc/core-darwin-arm64 \
        @swc/core-darwin-x64 \
        @swc/core-linux-arm-gnueabihf \
        @swc/core-linux-arm64-gnu \
        @swc/core-linux-arm64-musl \
        @swc/core-linux-x64-gnu \
        @swc/core-linux-x64-musl \
        @swc/core-win32-arm64-msvc \
        @swc/core-win32-ia32-msvc \
        @swc/core-win32-x64-msvc
```

Alternatively, you could use this Node script to merge the `optionalDependencies` from `@swc/core` into your own `package.json`:

```js
/* eslint-env node */

const path = require('path')
const fs = require('fs')

const optionalDepsFromSwcCore = require('@swc/core/package.json').optionalDependencies

if (!Object.keys(optionalDepsFromSwcCore).length) {
  console.log('@swc/core has no optional deps')
  process.exit(1)
}

console.log('@swc/core optional deps', optionalDepsFromSwcCore)

// merge @swc/core's optional deps with our own from package.json
const pkgJsonPath = path.resolve('package.json')
const pkgJson = require(pkgJsonPath)
const optionalDeps = {
  ...pkgJson.optionalDependencies,
  ...optionalDepsFromSwcCore,
}

// sort the optional deps by their keys
pkgJson.optionalDependencies = Object.keys(optionalDeps)
  .sort()
  .reduce((acc, key) => {
    acc[key] = pkgJson.optionalDependencies[key]
    return acc
  }, {})

console.log('merging optionalDependencies in', pkgJsonPath)
fs.writeFileSync(pkgJsonPath, JSON.stringify(pkgJson, null, 2) + '\n')
```

With those optional deps specified in your project's `package.json`, the lockfile will always contain entries for those deps even using the lockfile refresh method above. Only the dep that is supported on your platform would actually be installed.
