# webstorm-tsconfig-extends-exports-bug

The resolution of "extends" path in tsconfig.json doesn't account for the "exports" of package.json in workspaces.

1. `npm install`
2. Go to `a/src/index.ts`
3. Right click `someUtil`, Show context actions, Import ...

Expected:

The import from '@workspace/c/utils' is inserted.

Actual:

The import from '@workspace/c/src/utils' is inserted. Additionally, in `tsconfig.json`, the path in `extends` shows an error:

```
Cannot resolve file tsconfig.base.json 
```

## Explanation

This sample project uses npm workspaces (can be pnpm or yarn as well) and has three workspaces:

1. `a` - workspace that wants to import a util from `c`. It extends the `tsconfig.base.json` from `b` in its `tsconfig.json`.
2. `b` - workspace that defines a shared `tsconfig.base.json` file in its `src` folder and exports it via the `exports` field in `package.json` as `@workspace/b/tsconfig.base.json`.
3. `c` - workspace that exports a utility function from the `src/utils.ts` file via the `exports` field in `package.json` as `@workspace/c/utils`

The `moduleResolution` in `tsconfig.base.json` is set to `Bundler`, which should enable `exports` resolution according to [this](https://youtrack.jetbrains.com/issue/WEB-60536/Support-exports-field-of-package.json-for-TypeScript-files#focus=Comments-27-8290294.0-0) comment, but I guess, because resolving the path to the config requires `exports` resolution, it doesn't work.
With `exports` resolution, the `@workspace/b/tsconfig.base.json` and the import for `someUtil` should be resolved to `@workspace/c/utils`.
Though weirdly enough other options seem to be picked up - like the `"types": ["vite/client"]` option, which adds types for `import.meta.env`. I'm guessing this is the ts language service working correctly, while the imports are handled by the webstorm's own typescript service. The fact that running `npm run typecheck` works correctly confirms this theory.

Adding `"moduleResolution": "Bundler"` to the `tsconfig.json` of the workspaces does seem to solve the issue with imports,
but I don't think that makes webstorm resolve the path to `tsconfig.base.json` as well, because the `Cannot resolve file tsconfig.base.json` error still shows up.

Changing the path to `tsconfig.base.json` to `@workspace/b/src/tsconfig.base.json` fixes the issue with imports but breaks
the language service and `npm run typecheck` reports errors.

```
File '@workspace/b/src/tsconfig.base.json' not found.
```

This works correctly in vscode.

## Possible solution

It doesn't make sense to me why the typescript's module resolution is used to resolve paths in tsconfig and the ts compiler seems to agree with me. I think path resolution in tsconfig should just use node's path resolution, including `exports`, regardless of the options in the tsconfig.
