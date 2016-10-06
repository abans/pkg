# pkg

[![Build Status](https://travis-ci.com/zeit/pkg.svg?token=CPbpm6MRBVbWVmDFaLxs&branch=master)](https://travis-ci.com/zeit/pkg)

Package your Node.js project into an executable

## Use cases

* Make a commercial version of your application without sources
* Make a demo/evaluation/trial version of your app without sources
* Instantly make executables for other platforms (cross-compilation)
* Make some kind of self-extracting archive or installer
* No need to install Node.js and npm to deploy the packaged application
* No need to download hundreds of files via `npm install` to deploy
your application. Deploy it as a single file
* Put your assets inside the executable to make it even more portable
* Test your app against new Node.js version without installing it

## Install

```
npm install -g pkg
```

## CLI usage

Run `pkg --help` without arguments to see list of options.

The entrypoint of your project is a mandatory CLI argument.
It may be:

* Path to entry file. Suppose it is `/path/app.js`, then
packaged app will work the same way as `node /path/app.js`
* Path to `package.json`. `Pkg` will follow `bin` property of
the specified `package.json` and use it as entry file.
* Path to directory. `Pkg` will look for `package.json` in
the specified directory. See above.

### Targets

The final executables can generated for several target machines
at a time. You can specify the list of targets via `--targets`
option. A canonical target consists of 3 elements, separated by
dashes, for example `node6-macos-x64` or `node4-linux-armv6`:

* **nodeRange** node${n} or latest
* **platform** freebsd, linux, macos, win
* **arch** x64, x86, armv6, armv7

You may omit any element (and specify just `node6` for example).
The omitted elements will be taken from current platform or
system-wide Node.js installation (it's version and arch).
There is also an alias `host`, that means that all 3 elements
are taken from current platform/Node.js. By default targets are
`linux,macos,win` for current Node.js version and arch.

### Config

During packaging process `pkg` parses your sources, detects
calls to `require`, traverses the dependencies of your project
and includes them into final executable. In most cases you
don't need to specify anything manually. However your code
may have `require(variable)` calls (so called non-literal
argument to `require`) or use non-javascript files (for
example views, css, images etc).
```
  require('./build/' + cmd + '.js')
  path.join(__dirname, 'views/' + viewName)
```
Such cases are not handled by `pkg`. So you must specify the
files - scripts and assets - manually in a config. It is
recommended to use package.json's `pkg` property.
```
  "pkg": {
    "scripts": "build/**/*.js",
    "assets": "views/**/*"
  }
```


### Scripts

`scripts` is a [glob](https://github.com/sindresorhus/globby)
or list of globs. Files specified as `scripts` will be compiled
using `v8::ScriptCompiler` and placed into executable without
sources. They must conform JS standards of those Node.js versions
you target (see [Targets](#targets)), i.e. be already transpiled.

### Assets

`assets` is a [glob](https://github.com/sindresorhus/globby)
or list of globs. Files specified as `assets` will be packaged
into executable as raw content without modifications. Javascript
files may be specified as `assets` as well. Their sources will
not be stripped. It improves performance of execution of those
files and simplifies debugging.

See also
[Detecting assets in source code](#detecting-assets-in-source-code) and
[Virtual filesystem](#virtual-filesystem).

### Options

Node.js application can be called with runtime options
(belonging to Node.js or V8). To list them type `node --help` or
`node --v8-options`. You can "bake" these runtime options into
packaged application. The app will always run with the options
turned on. Just remove `--` from option name.
```
pkg app.js --options expose-gc
```

### Output

You may specify `--output` if you create only one executable
or `--out-dir` to place executables for multiple targets.

### Debug

Pass `--debug` to `pkg` to get a log of packaging process.
If you have issues with some particular file (seems not packaged
into executable), it may be useful to look through the log.

### Build

`pkg` has so called `base binaries` - they are actually same
`node` executables but with some patches applied. They are
used as a base for every executable `pkg` creates. `pkg`
downloads precompiled base binaries before packaging your
application. If you prefer to compile base binaries from
source instead of downloading them, you may pass `--build`
option to `pkg`. First ensure your computer meets the
requirements to compile original Node.js:
[BUILDING.md](https://github.com/nodejs/node/blob/master/BUILDING.md)

## Usage of packaged app

Command line call to packaged app `./app a b` is equivalent
to `node app.js a b`

## Virtual filesystem

During packaging process `pkg` collects project files and places
them into final executable. At run time the packaged application has
internal virtual filesystem where all that files reside.

Packaged VFS files have `/thebox/` prefix in their paths (or
`C:\thebox\` in Windows). If you used `pkg /path/app.js` command line,
then `__filename` value will be likely `/thebox/path/app.js`
at run time. `__dirname` will be `/thebox/path` as well. Here is
the comparison table of path-related values:

value                          | with `node`         | packaged                 | comments
-------------------------------|---------------------|--------------------------|-----------
__filename                     | /project/app.js     | /thebox/project/app.js   |
__dirname                      | /project            | /thebox/project          |
process.cwd()                  | /project            | /deploy                  | suppose the app is called ...
process.execPath               | /usr/bin/nodejs     | /deploy/app-x64          | `app-x64` and run in `/deploy`
process.argv[0]                | /usr/bin/nodejs     | /deploy/app-x64          |
process.argv[1]                | /project/app.js     | /deploy/app-x64          |
process.pkg.entrypoint         | undefined           | /thebox/project/app.js   |
process.pkg.defaultEntrypoint  | undefined           | /thebox/project/app.js   |
require.main.filename          | /project/app.js     | /thebox/project/app.js   |

Hence in order to make use of the file collected at packaging
time (pick up own JS plugin or serve an asset) you should take
`__filename`, `__dirname`, `process.pkg.defaultEntrypoint`
or `require.main.filename` as a base for your path calculations.
One way is just `require` or `require.resolve` because they use
current `__dirname` by default. But they are applicable to
javascript files only. For assets use
`path.join(__dirname, '../path/to/asset')`. Learn more about
`path.join` in
[Detecting assets in source code](#detecting-assets-in-source-code).

On the other hand, in order to access real file system (pick
up a user's JS plugin or list user's directory) you should take
`process.cwd()` or `path.dirname(process.argv[1])`. Why `argv[1]`?
Because you will be able to run the project both with `node`
and in packaged state.

## Detecting assets in source code

When `pkg` encounters `path.join(__dirname, '../path/to/asset')`,
it automatically packages the file specified as an asset. See
[Assets](#assets). Pay attention that `path.join` must have two
arguments and the last one must be a string literal.

This way you may even avoid creating `pkg` config for your project.
