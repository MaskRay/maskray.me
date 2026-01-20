---
layout: post
title: Layering check with Clang
author: MaskRay
tags: [clang]
---

Updated in 2023-07.

This article describes some [Clang header modules](https://clang.llvm.org/docs/Modules.html) features that apply to `#include`.
These features enforce a more explicit dependency graph, which provide documentation purposes and makes refactoring convenient.
The benefits of clean header inclusions are well described in [Include What You Use](https://www.fluentcpp.com/2021/01/01/include-what-you-use/) as well, so I won't repeat them here.

When using C++20 modules, these features apply to `#include` in a global module fragment (`module;`) but have no effect for import declarations.

## Layering check

### `-fmodules-decluse`

For a `#include` directive, this option emits an error if the following conditions are satisfied (see `clang/lib/Lex/ModuleMap.cpp` `diagnoseHeaderInclusion`):

* The main file is within a module (called "source module", say, `A`).
* The main file or an included file from the source module includes a file from another module `B`.
* `A` does not have a use-declaration of `B` (no `use B`).

For the first condition, `-fmodule-map-file=` is needed to load the source module map and `-fmodule-name=A` is needed to indicate that the source file is logically part of module `A`.

For the second condition, the module map defining `B` must be loaded by specifying `-fimplicit-module-maps` (implied by `-fmodules` and `-fcxx-modules`) or a `-fmodule-map-file=`.

<!-- more -->

Here is an example:

```sh
cat > a.cc <<'eof'
#include "dir/b.h"
#include "dir/c.h"
int main() {}
eof
echo 'module A { header "a.h" use B }' > module.modulemap
mkdir -p dir
echo '#include "c.h"' > dir/b.h
echo 'void fc();' > dir/c.h
cat > dir/module.modulemap <<'eof'
module B { header "b.h" use C }
module C { header "c.h" }
eof
```

The following commands lead to an error about `dir/c.h`. `#include "dir/b.h"` is allowed because module `A` has a use-declaration on module `B`.

```text
% clang -fsyntax-only -fmodules-decluse -fmodule-map-file=module.modulemap -fmodule-name=A -fimplicit-module-maps a.cc
a.cc:1:10: error: module A does not depend on a module exporting 'dir/c.h'
#include "dir/c.h"
         ^
1 error generated.
% clang -fsyntax-only -fmodules-decluse -fmodule-map-file=module.modulemap -fmodule-name=A -fmodule-map-file=dir/module.modulemap a.cc
a.cc:1:10: error: module A does not depend on a module exporting 'dir/c.h'
#include "dir/c.h"
         ^
1 error generated.
```

`textual header "c.h"` triggers the error as well.

If we remove `-fmodule-name=A`, we won't see an error: Clang does not know `a.cc` logically belongs to module `A`.

### `-fmodules-strict-decluse`

This is a strict variant of `-fmodules-decluse`. If an included file is not within a module, `-fmodules-decluse` allows the inclusion while `-fmodules-strict-decluse` reports an error.

Use the previous example, but drop `-fimplicit-module-maps` and `-fmodule-map-file=dir/module.modulemap` so that Clang thinks `dir/c.h` is not within a module.
Let's see the distinction between `-fmodules-decluse` and `-fmodules-strict-decluse`.

```text
% clang -fsyntax-only -fmodules-decluse -fmodule-map-file=module.modulemap -fmodule-name=A a.cc
% clang -fsyntax-only -fmodules-strict-decluse -fmodule-map-file=module.modulemap -fmodule-name=A a.cc
a.cc:1:10: error: module A does not depend on a module exporting 'dir/c.h'
#include "dir/c.h"
         ^
1 error generated.
```

Many systems do not ship Clang module map files for C/C++ standard libraries, so `-fmodules-strict-decluse` is not suitable.
```
% clang -fsyntax-only -fmodules-strict-decluse -fmodule-map-file=module.modulemap -fmodule-name=A -fimplicit-module-maps a.cc
a.cc:1:10: error: module A does not depend on a module exporting 'stdio.h'
#include <stdio.h>
         ^
...
```

In Bazel, `tools/cpp/generate_system_module_map.sh` generates a module map listing all system headers.

### `-Wprivate-header`

In the Clang [module map language](https://clang.llvm.org/docs/Modules.html#module-map-language), a header with the private specifier cannot be included from outside the module itself.
`-Wprivate-header` is an enabled-by-default warning enforcing this rule. The warning is orthogonal to `-fmodules-decluse`/`-fmodules-strict-decluse`.

To see its effect, change `dir/module.modulemap` by making `b.h` private:
```text
module B { private header "b.h" use C }
module C { header "c.h" }
```

Then `clang -fsyntax-only -fmodule-map-file=module.modulemap -fmodule-name=A -fimplicit-module-maps a.cc` will report an error:
```text
a.cc:2:10: error: use of private header from outside its module: 'dir/c.h' [-Wprivate-header]
#include "dir/c.h"
         ^
1 error generated.
```

## Relation with include-what-you-use and dependency-related linker options

To make full power of the layering check features, the source files must have clean header inclusions.

In the following example, `a.cc` gets `dir/c.h` declarations transitively via `dir/b.h` but does not include `dir/b.h` directly.
`-fmodules-strict-decluse` cannot flag this case.

```sh
cat > a.cc <<'eof'
#include "dir/b.h"
int main() { fc(); }
eof
echo 'module A { header "a.h" use B }' > module.modulemap
mkdir -p dir
echo '#include "c.h"' > dir/b.h
echo 'void fc();' > dir/c.h
cat > dir/module.modulemap <<'eof'
module B { header "b.h" use C }
module C { header "c.h" }
eof
```

If `#include "dir/b.h"` is added due to clean header inclusions, `-fmodules-decluse` will report an error.

In the absence of clean header inclusions, [dependency-related linker options](/blog/2021-06-13-dependency-related-linker-options) (`-z defs`, `--no-allow-shlib-undefined`, and `--warn-backrefs`) can mitigate some brittle build problems.

With C++20 modules, if `B` does not export-import `C`, `a.cc` cannot get `C`'s declarations from `import B;`.

## Build system support

Bazel is the first build system implementing the layering check feature and as of today the only build system.

### Bazel

Bazel has implemented the built-in feature `layering_check` (<https://github.com/bazelbuild/bazel/pull/11440>) using both `-fmodules-strict-decluse` and `-Wprivate-header`.

Bazel generates `.cppmap` module files from `deps` attributes.
`hdrs` and `textual_hdrs` files are converted to `textual header` declarations while `srcs` headers are converted to `private textual header` declarations.
`deps` attributes are converted to use declarations.

When building a target with Clang and `layering_check` is enabled for the target, Bazel passes a list of `-fmodule-map-file=` (according to the build target and its direct dependencies) and `-fmodule-name=`  to Clang.

```sh
touch ./WORKSPACE
cat > ./a.cc <<'eof'
#include "a.h"
int main() { fc(); }
eof
echo '#include "c.h"' > ./a.h
echo '#include "c.h"' > ./b.h
cat > ./BUILD <<'eof'
cc_binary(
    name = "a",
    srcs = ["a.cc", "a.h"],
    deps = [":b"],  # :c is missing
)

cc_library(
    name = "b",
    hdrs = ["b.h"],
    deps = [":c"],
)

cc_library(
    name = "c",
    srcs = ["c.cc"],
    hdrs = ["c.h"],
)
eof
echo 'void fc() {}' > ./c.cc
echo 'void fc();' > ./c.h
```

The following build command gives an error with Clang 16.
The main file `a.cc` includes `a.h` whose inclusion of `c.h` does not have a corresponding use declaration.
(Clang before 16.0 did not check `-fmodules-decluse` for the non-main-file textual header `a.h`: <https://reviews.llvm.org/D132779>)

```sh
% CC=/tmp/Rel/bin/clang bazel build --features=layering_check :a
INFO: Analyzed target //:a (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
ERROR: /tmp/f/BUILD:1:10: Compiling a.cc failed: (Exit 1): clang-16 failed: error executing command /tmp/Rel/bin/clang-16 -U_FORTIFY_SOURCE -fstack-protector -Wall -Wthread-safety -Wself-assign -Wunused-but-set-parameter -Wno-free-nonheap-object -fcolor-diagnostics -fno-omit-frame-pointer '-std=c++0x' ... (remaining 29 arguments skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging
In file included from a.cc:1:
./a.h:1:10: error: module //:a does not depend on a module exporting 'c.h'
#include "c.h"
         ^
1 error generated.
```

The relevant Clang driver options are: `'-fmodule-name=//:a' '-fmodule-map-file=bazel-out/k8-fastbuild/bin/a.cppmap' -fmodules-strict-decluse -Wprivate-header '-fmodule-map-file=external/local_config_cc/module.modulemap' '-fmodule-map-file=bazel-out/k8-fastbuild/bin/b.cppmap'`.
Note that `c.cppmap` is not loaded as `:c` is not a direct dependency.

Here are the generated `.cppmap` module maps:
```text
% cat bazel-out/k8-fastbuild/bin/a.cppmap
module "//:a" {
  export *
  private textual header "../../../a.h"
  use "//:b"
  use "@bazel_tools//tools/cpp:malloc"
  use "crosstool"
}
extern module "//:b" "../../../bazel-out/k8-fastbuild/bin/b.cppmap"
extern module "@bazel_tools//tools/cpp:malloc" "../../../bazel-out/k8-fastbuild/bin/external/bazel_tools/tools/cpp/malloc.cppmap"
extern module "crosstool" "../../../external/local_config_cc/module.modulemap"
% cat bazel-out/k8-fastbuild/bin/b.cppmap
module "//:b" {
  export *
  textual header "../../../b.h"
  use "//:c"
  use "crosstool"
}
extern module "//:c" "../../../bazel-out/k8-fastbuild/bin/c.cppmap"
extern module "crosstool" "../../../external/local_config_cc/module.modulemap"
% cat bazel-out/k8-fastbuild/bin/c.cppmap
module "//:c" {
  export *
  textual header "../../../c.h"
  use "crosstool"
}
extern module "crosstool" "../../../external/local_config_cc/module.modulemap"
```

`external/local_config_cc/module.modulemap` contains files in Clang's default include paths to make `-fmodules-strict-decluse` happy.

#### `parse_headers`

The feature `parse_headers` ensures that header files are self-contained.
When this feature is configured and enabled, Bazel parses non-textual headers in the `srcs` and `hdrs` attributes by compiling them as main files using `-fsyntax-only`.
These headers do not require other headers or macros as a precondition.
A source file including such a header can freely reorder it among the included files.

Some problems detected by `parse_headers` can also be detected by `layering_check`. In other words, even if `parse_headers` is disabled, `layering_check` can still detect these problems.

When `layering_check` is enabled, it applies to the header compiles that occur due to `parse_headers`. Enabling both features help detect more problems.

## Application

llvm-project has an unsupported (in the sence that contributors are not required to fix it) Bazel build system at `utils/bazel/`.
`features = ["layering_check"]` is enabled for [llvm/, clang/](https://reviews.llvm.org/D141553), and [mlir/](https://reviews.llvm.org/D113952).

## Relation with C++20 modules

Both the module `export` keyword and the layering check features can limit the harm of transitive inclusion.
The module `export` keyword makes a library explicit about what APIs it exports, while the layering check feature makes the user explicit what APIs it uses.
