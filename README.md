# dts-buddy

A tool for creating `.d.ts` bundles.

## Why?

If you're creating a package with subpackages (i.e. you can import from `my-lib` but also `my-lib/subpackage`), correctly exposing types to your users in a way that works everywhere is [difficult difficult lemon difficult](https://www.youtube.com/watch?v=7mAFiPVs3tM).

The TypeScript team recommends a [variety of strategies](https://github.com/andrewbranch/example-subpath-exports-ts-compat/tree/main) but they all involve adding a bunch of otherwise useless files to your package.

One thing that works everywhere is `declare module` — if you expose a file like this...

```ts
declare module 'my-lib' {
	/**
	 * Add two numbers
	 */
	export function add(a: number, b: number): number;
}

declare module 'my-lib/subpackage' {
	/**
	 * Multiply two numbers
	 */
	export function multiply(a: number, b: number): number;
}
```

...then everyone will get autocompletion and typechecking for those functions. For bonus points, it should include a `.d.ts.map` file that allows 'go to definition' to take you to the original source. (This rules out hand-authoring the file, which you shouldn't be doing anyway.)

There are other benefits to this approach — you end up with smaller packages, and TypeScript has less work to do on startup, making everything quicker for your users.

Unfortunately, I couldn't find a tool for generating `.d.ts` bundles, at least not one that worked. `dts-buddy` aims to fill the gap.

## But really, why?

For [SvelteKit](https://kit.svelte.dev), where for a long time we hand-authored an `ambient.d.ts` file containing `declare module` blocks for subpackages, and _also_ had an `index.d.ts` file for the main types that had to duplicate the definitions of certain functions. Every time we changed anything, we had to update things in multiple places, and contributing to the codebase was unnecessarily difficult.

An extra dimension is that we have virtual modules like `$app/environment`, which can't be expressed using any of the techniques suggested by the TypeScript team.

`dts-buddy` means we can automate generation of all our type definitions, using the source as the, well, source of truth.

## How do I use it?

Add a script like this to your project:

```js
// scripts/generate-dts-bundle.js
import { createBundle } from 'dts-buddy';

await createBundle({
	project: 'tsconfig.json',
	output: 'types/index.d.ts',
	modules: {
		'my-lib': 'src/index.js',
		'my-lib/subpackage': 'src/subpackage.js'
	}
});
```

Then, inside your package.json:

```diff
{
	"name": "my-lib",
	"version": "1.0.0",
	"type": "module",
+	"types": "./types/index.d.ts",
	"files": [
		"src",
+		"types"
	],
	"exports": {
		".": {
+			"types": "./types/index.d.ts",
			"import": "./src/index.js"
		},
		"./subpackage": {
+			"types": "./types/index.d.ts",
			"import": "./src/subpackage.js"
		}
	},
	"scripts": {
+		"prepublishOnly": "node scripts/generate-dts-bundle.js"
	}
}
```

## Known limitations

This is a very rough-and-ready tool, that probably won't work for your use case (at least yet). In particular:

- everything gets exported, including your internal types. oops!
- names are not deconflicted. if you have two things called `Foo` declared in separate modules, they might clobber each other depending on where they get used. oops!
- no sourcemaps yet

## License

MIT