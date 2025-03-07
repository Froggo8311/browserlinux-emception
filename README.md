# Emception
Compile C/C++ code with [Emscripten](https://emscripten.org/) in the browser.
You can see it in action in the [live demo](https://jprendes.github.io/emception/).

# Build
To build Emception, simply clone the repo, and run `./build-with-docker.sh`. I've only built it on Linux, but you should be able to build it from anywhere you can run Docker. Bear in mind that the build will take a long while (specially cloning/building llvm and building pyodide), so go get a a cup of coffee.
```bash
git clone https://github.com/jprendes/emception.git
cd emception
./build-with-docker.sh
```

This will generate the files in `build/emception`. To build the demo, from the `demo` folder run `npm install` and `npm run build`. You can then run the demo locally running `npx serve build/demo`.
```bash
pushd demo
npm install
npm run build
popd
npx serve build/demo
```
# Interesting sub-projects
This is a list of parts of this project that I think could be interesting on their own.

## `llvm-box`
This is clang/llvm compiled to WebAssembly. It bundles many llvm executables in one binary, kind of like BusyBox bundles Unix command line utilities in one binary. `llvm-box` packs the following executables: `clang`, `lld`, `llvm-nm`, `llvm-ar`, `llvm-objcopy`, and `llc`.

This is done to reduce the binary size. With all the executables living in the same binary, they can share the same standard libraries as well as the common llvm codebase.

## `binaryen-box`
Just like with `llvm-box`, `binaryen-box` bundles many binaryen executables in one WebAssembly binary. `binaryen-box` packs the following executables: `wasm-as`, `wasm-ctor-eval`, `wasm-emscripten-finalize`, `wasm-metadce`, and `wasm-opt wasm-shell`.

As with `llvm-box`, this is done to reduce binary size.

## `wasm-transform`
This is the tool used to bundle many executables into one binary module. `wasm-transform` modifies the wasm object files from different executables so that they can be linked together into one binary. It also prints out the metadata required to create a new `main` function that dispatches execution to the different binaries.

`wasm-transform` applies the following changes on WebAssembly object files:
* renames the `main` symbol, appending a hash computed from the object file path.
* identifies the functions that run global constructors (called init functions) and renames them so that they are valid `C` function names and appends a hash computed from the object file path. Then it marks them as normal functions and removes them from the list of init functions.

Additionally, for each renamed symbol it prints:
* a line starting with `entrypoint` followed by the new name of the renamed `main` function.
* a line starting with `constructor` followed by a number representing the priority of the init function, and the new name of the renamed init function. Init functions should be run by priority in ascending order.

The information printed by `wasm-transform` can be used to create a new `main` function that dispatches execution to the renamed `main` functions, e.g., based on the value of `argv[0]`.

For example, bundling `executable_a` and `executable_b` would look like this:
```bash
$ wasm-transform executable_a.o executable_a_transformed.o
constructor 65535 _GLOBAL__sub_I_executable_a_cpp_123
entrypoint main_123
$ wasm-transform executable_b.o executable_b_transformed.o
constructor 65535 _GLOBAL__sub_I_executable_b_cpp_456
entrypoint main_456
```
With that information, the dispatching `main.cpp` function could look like this:
```cpp
#include <string_view>

extern "C" void _GLOBAL__sub_I_executable_a_cpp_123();
extern "C" void main_123();

extern "C" void _GLOBAL__sub_I_executable_b_cpp_456();
extern "C" void main_456();

int main(int argc, char const ** argv) {
    std::string_view const argv0 = argv[0];
    if (argv0.ends_with("executable_a")) {
        _GLOBAL__sub_I_executable_a_cpp_123();
        return main_123(argc, argv);
    } else if (argv0.ends_with("executable_b")) {
        _GLOBAL__sub_I_executable_b_cpp_456();
        return main_456(argc, argv);
    }
    return 1;
}
```
Everything can then be linked together:
```
em++ dispatching_main.cpp executable_a_transformed.o executable_b_transformed.o -o bundled_executables.mjs
```

## `wasm-package`
A tar-like application, that can pack several files in one archive. It preserves file permissions and symlinks. Its main purpose is to pre-load files in an Emscripten module, using a native build to pack the files, and a WebAssembly build to unpack them.

This works particularly well in conjunction with a `brotli` build in WebAssembly for servers that don't serve `brotli` compressed content (like GitHub pages): Create a `brotli` module, a `wasm-package` module, and your target module, all sharing the same file system using `SHAREDFS.js` (see below). Write the compressed file in a temporary location and use the `brotli` build to decompress it. Then use the `wasm-package` to unpack the archive content. Now your module should be able to access the preloaded files.

## `SHAREDFS.js`
This tool can be used to share a file system between different Emscripten modules. The whole filesystem is shared between the modules, except for `/dev` and `/proc` since they contain process specific files/devices.

It can be used as follow:
```js
import Module from "./Module.mjs"; // the Emscripten module
import shareFS from "./SHAREDFS.js";

const mod_1 = {};
await new Module(mod_1);

const mod_2 = {
    preInit: () => shareFS(mod_1.FS, mod_2.FS)
};
await new Module(mod_2);

mod_1.writeFile("/tmp/test", "hello world!");
const str = mod_2.readFile("/tmp/test", { encoding: "utf8" });
console.log(str); // logs "hello world!"
```
