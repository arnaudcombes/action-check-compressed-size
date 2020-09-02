# action-check-compressed-size

A GitHub action that reports changes in compressed file sizes on your PRs.

**Heavily inspired by https://github.com/preactjs/compressed-size-action plus:**  
**- rewritten in Typescript 🎉**  
**- correct calculation of % differences**  
**- can install packages in a subdirectory**  
**- allow to normalize both js and css files**  

**Used in production at [Arhia](https://github.com/Arhia)**  

- Automatically uses `yarn` or `npm ci` when lockfiles are present
- Builds your PR, then builds the target and compares between the two
- Doesn't upload anything or rely on centralized storage
- Supports [custom build scripts](#customizing-the-build) and [file patterns](#customizing-the-list-of-files)


<img width="600" src="https://user-images.githubusercontent.com/7466144/91588082-eb759680-e957-11ea-8655-23d7fab71217.png">

### Usage:

Add a workflow (`.github/workflows/main.yml`):

```yaml
name: Compressed Size
on: [pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: Arhia/action-check-compressed-size@v0.7
      with:
        repo-token: "${{ secrets.GITHUB_TOKEN }}"
```

### Customizing the Build

By default, this action will try to build your PR by running the `"build"` [npm script](https://docs.npmjs.com/misc/scripts) in your `package.json`.

If you need to perform some tasks after dependencies are installed but before building, you can use a "postinstall" npm script to do so. For example, in Lerna-based monorepo:

```json
{
  "scripts": {
    "postinstall": "lerna bootstrap",
    "build": "lerna run build"
  }
}
```

It is also possible to define a `"prebuild"` npm script, which runs after `"postinstall"` but before `"build"`.

You can also specify a completely different [npm script](https://docs.npmjs.com/misc/scripts) to run instead of the default (`"build"`). To do this, add a **`build-script` option** to your `yml` workflow:

```diff
name: Compressed Size

on: [pull_request]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - uses: Arhia/action-check-compressed-size@v0.7
      with:
        repo-token: "${{ secrets.GITHUB_TOKEN }}"
+       build-script: "ci"
```

### Customizing working directory

`directory` option allow to run all the tasks in a subfolder.  
It's only convenient if all your stuff is in a subdirectory of your git repository.  

For instance, if `package.json` is in the subfolder `client/`

Example : 

```yaml
  directory: "client/"
```

### Customizing the list of files

`action-check-compressed-size` defaults to tracking the size of all JavaScript files within `dist/` directories - anywhere in your repository, not just at the root. You can change the list of files to be tracked and reported using the `pattern` and `exclude` options, both of which are [minimatch patterns](https://github.com/motemen/minimatch-cheat-sheet/blob/master/README.md):

```diff
name: Compressed Size
on: [pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: Arhia/action-check-compressed-size@v0.7
      with:
        repo-token: "${{ secrets.GITHUB_TOKEN }}"
+       pattern: "./build-output/**/*.{js,css,html,json}"
+       exclude: "{./build-output/manifest.json,**/*.map,**/node_modules/**}"
```

Files are collected by finding matches for `pattern`, then any of those that match `exclude` are ignored. For that reason, most project don't need to modify `exclude`. The default values for `pattern` and `exclude` are as follows:

```yaml
with:
  # Any JS files anywhere within a dist directory:
  pattern: "**/dist/**/*.js"

  # Always ignore SourceMaps and node_modules:
  exclude: "{**/*.map,**/node_modules/**}"
```

### Dealing with hashed filenames

`strip-hash` option allows passing an array of Regular Expression pattern that will be used to remove hashes from filenames. The un-hashed filenames are used both for size comparison and display purposes.

By default, the characters matched by the regex are removed from filenames.
In the example below, a filename `foo.abcde.js` will be converted to `foo.js`:

```yaml
  strip-hash: '["\\b\\w{5}\\."]'
```

This can be customized further using parens to create submatches, which mark where a hash occurs. When a submatch is detected, it will be replaced with asterisks. This is particularly useful when mix of hashed and unhashed filenames are present.
In the example below, a filename `foo.abcde.chunk.js` will be converted to `foo.*****.chunk.js`:

```yaml
  strip-hash: '["\\.(\\w{8})\\.chunk\\.js$", "precache-manifest\\.(\\w{32})\\.js$", "runtime-main\\.(\\w{8})\\.js$", "\\.(\\w{8})\\.chunk\\.css$"]'

```

### Increasing the required threshold

By default, a file that's been changed by a single byte will be reported as changed. If you'd prefer to require a certain minimum threshold for a file to be changed, you can specify `minimum-change-threshold` in bytes:

```yaml
  minimum-change-threshold: 100
```

In the above example, a file with a delta of less than 100 bytes will be reported as unchanged.
