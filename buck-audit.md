# buck audit

Provide information about build configuration parameters, targets, and rules.
Syntax:

```
buck audit <command> [ <parameter> . . . ] <target>  . . .
```

Example:

```
buck audit input //java/com/example/app:amazing
```

For more examples, see the command descriptions and **Examples** section below.

## Commands

* `alias --list`
    List the aliases declared in [`.buckconfig`](https://buck.build/files-and-dirs/buckconfig.html) and `.buckconfig.local`. This command lists only the aliases, not their values. To see the values, use the `buck audit config` command.
* `buck audit cell`
    List the absolute paths to the cells that are specified in the [`[repositories]`](https://buck.build/files-and-dirs/buckconfig.html#repositories) section of the `.buckconfig` file that is in the root of the current cell. The path to each cell is prefixed with the specified alias for that cell. For example:

```
$ buck audit cell 
buck: /Users/buxster/local/buck
bazel_skylib: /Users/buxster/local/buck/third-party/skylark/bazel-skylib
```

(In this example, `buxster` is the name of the current user.)
If you specify the `--paths-only` parameter, Buck outputs only the absolute paths to the cells, without the aliases.

```
$ buck audit cell --paths-only
/Users/buxster/local/buck
/Users/buxster/local/buck/third-party/skylark/bazel-skylib
```

If your `.buckconfig` does not contain a `[repositories]` section, then `buck audit cell` doesn't return any output.

* `classpath <targets>`
    List the Java classpath used to run the specified targets. This does not work for all build rule types.
* `config {<section> | <section.property>} [...]`
    List the values from `.buckconfig` (and `.buckconfig.local`) for the specified sections and properties.
    If you specify only the section name, `buck audit config` lists all the properties and values for that section.
    Note that properties and values specified with [`--config`](https://buck.build/command/common_parameters.html) are not surfaced by this command, and those properties and values override both `.buckconfig` and `.buckconfig.local`.
    Use `--tab` to get tab-delimited output.
    Example: To get the C compiler and the C++ compiler, use

```
buck audit config cxx.cc cxx.cxx
[cxx]
    cc = /usr/bin/gcc
    cxx = /usr/bin/g++
```

or (with `--tab`)

```
buck audit config --tab cxx.cc cxx.cxx
cxx.cc    /usr/bin/gcc
cxx.cxx    /usr/bin/g++
```

* `dependencies <targets>`
    List the dependencies used to build the specified targets. Results are listed in alphabetical order. By default, only direct dependencies are listed; to show transitive dependencies, use the `--transitive` parameter. To show tests for a rule, use the `--include-tests` parameter. This prints out a rule's tests as if they were dependencies of the rule. To print out all of the *test's* dependencies as well, combine `--include-tests` with the `--transitive` parameter.
* `flavors <targets>`
    List the flavors that are available for the specified targets and what the default flavor is for each target. If the `flavors` command prints `no flavors`, it indicates that, although the target rule supports flavors, Buck was not able to extract any. If the `flavors` command prints `unknown`, it indicates that the target rule doesn't support flavors. The `flavors` command supports the `--json` parameter for JSON-formatted output.
* `input <targets>`
    List the input source and resource files used to build the specified targets.
* `includes <build_file>`
    List the [build file](https://buck.build/concept/build_file.html)s, and their extensions, that are included in the specified build file.
* `modules`
    List the Java modules known by Buck as well as their content hashes and dependencies.
* `ruletype <rule>`
    Print the Python signature for the specified rule.
    The following command line uses `buck audit ruletype` to view the arguments supported by the [`remote_file`](https://buck.build/rule/remote_file.html) rule.

```
buck audit ruletype remote_file
def remote_file (
    name,
    sha1,
    url,
    labels = None,
    licenses = None,
    out = None,
    type = None,
):
    ...
```

* `ruletypes`
    List all the rules that Buck supports, in alphabetical order.
    **Example**
    The following example prints *all* the rules that Buck supports. Note that the output is truncated for brevity.

```
buck audit ruletypes
android_aar
android_app_modularity
android_binary
android_build_config
android_bundle
android_instrumentation_apk
android_instrumentation_test
android_library
android_manifest
android_prebuilt_aar
android_resource
apk_genrule
apple_asset_catalog
apple_binary
<truncated>
```

**Example**
The following command line uses `buck audit ruletypes` with the `grep` command to print all the build rules that have the string `android` in their names.

```
buck audit ruletypes | grep android
```

Note that these are not all the rules that Buck provides for Android development. For example, the rules `apk_genrule` and `ndk_library` support Android development, but do not themselves contain the string `android` in their names.

* `tests <targets> [...]`
    List the tests for the specified targets. Results are listed in alphabetical order. Only tests for the specified targets are printed, though multiple targets may be specified on a single command line. This command is intended to be used in conjunction with the `audit dependencies` command. For example, to retrieve a list of all tests for a given project, use:

```
buck audit dependencies --transitive PROJECT | xargs buck audit tests
```

## Parameters

* `--include-tests`
    Show the tests for the specified targets. Can be combined with the `--transitive` parameter. For more information, see the `dependencies` command.
* `--json`
    Output the results as JSON.
* `--list`
    List `.buckconfig` and `.buckconfig.local` aliases. Used only with the `aliases` command. For more information, see that command.
* `--tab`
    Output the results using tab delimiters. Used only with the `config` command. For more information, see that command.
* `--transitive`
    Show transitive dependencies in addition to direct dependencies. Can be combined with the `--include-tests` parameter. For more information, see the `dependencies` command.

## Examples

```
## BUCK## For all of the following examples, assume this BUCK file exists in# the `examples` directory.#
java_library(
  name = 'one',
  srcs = [ '1.txt' ],
  deps = [':two',':three',],)

java_library(
  name = 'two',
  srcs = [ '2.txt' ],
  deps = [':four',],)

java_library(
  name = 'three',
  srcs = [ '3.txt' ],
  deps = [':four',':five',],)

java_library(
  name = 'four',
  srcs = [ '4.txt' ],
  deps = [':five',],)

java_library(
  name = 'five',
  srcs = [ '5.txt' ],)
```

List all of the source files used to build the `one` library

```
buck audit input //examples:one
examples/1.txt
examples/2.txt
examples/3.txt
examples/4.txt
examples/5.txt
```

Output a JSON representation of all of the source files used to build the `two` library. In this JSON object, each key is a build target and each value is an array of the source paths used to build that rule.

```
buck audit input --json //examples:two
{"//examples:two": ["examples/2.txt"],"//examples:four": ["examples/4.txt"],"//examples:five": ["examples/5.txt"],}
```

List all of the rules that the `one` library directly depends on

```
buck audit dependencies //examples:one
//examples:three
//examples:two
```

List all of the rules that the `one` library transitively depends on

```
buck audit dependencies --transitive //examples:one
//examples:five
//examples:four
//examples:three
//examples:two
```

Output a JSON representation of all of the rules that the `two` library transitively depends on.

```
buck audit dependencies --transitive --json //examples:two
{"//examples:two": ["//examples:five","//examples:four"]}
```

Output a JSON representation of the direct dependencies of the `two` and `three` libraries.

```
buck audit dependencies --json //examples:two //examples:three
{"//examples:three": ["//examples:five","//examples:four"],"//examples:two": ["//examples:four"]}
```



Provide information about build configuration parameters, targets, and rules.
Syntax:

```
buck audit <command> [ <parameter> . . . ] <target>  . . .
```

Example:

```
buck audit input //java/com/example/app:amazing
```

For more examples, see the command descriptions and **Examples** section below.

## Commands

* `alias --list`
    List the aliases declared in [`.buckconfig`](https://buck.build/files-and-dirs/buckconfig.html) and `.buckconfig.local`. This command lists only the aliases, not their values. To see the values, use the `buck audit config` command.
* `buck audit cell`
    List the absolute paths to the cells that are specified in the [`[repositories]`](https://buck.build/files-and-dirs/buckconfig.html#repositories) section of the `.buckconfig` file that is in the root of the current cell. The path to each cell is prefixed with the specified alias for that cell. For example:

```
$ buck audit cell 
buck: /Users/buxster/local/buck
bazel_skylib: /Users/buxster/local/buck/third-party/skylark/bazel-skylib
```

(In this example, `buxster` is the name of the current user.)
If you specify the `--paths-only` parameter, Buck outputs only the absolute paths to the cells, without the aliases.

```
$ buck audit cell --paths-only
/Users/buxster/local/buck
/Users/buxster/local/buck/third-party/skylark/bazel-skylib
```

If your `.buckconfig` does not contain a `[repositories]` section, then `buck audit cell` doesn't return any output.

* `classpath <targets>`
    List the Java classpath used to run the specified targets. This does not work for all build rule types.
* `config {<section> | <section.property>} [...]`
    List the values from `.buckconfig` (and `.buckconfig.local`) for the specified sections and properties.
    If you specify only the section name, `buck audit config` lists all the properties and values for that section.
    Note that properties and values specified with [`--config`](https://buck.build/command/common_parameters.html) are not surfaced by this command, and those properties and values override both `.buckconfig` and `.buckconfig.local`.
    Use `--tab` to get tab-delimited output.
    Example: To get the C compiler and the C++ compiler, use

```
buck audit config cxx.cc cxx.cxx
[cxx]
    cc = /usr/bin/gcc
    cxx = /usr/bin/g++
```

or (with `--tab`)

```
buck audit config --tab cxx.cc cxx.cxx
cxx.cc    /usr/bin/gcc
cxx.cxx    /usr/bin/g++
```

* `dependencies <targets>`
    List the dependencies used to build the specified targets. Results are listed in alphabetical order. By default, only direct dependencies are listed; to show transitive dependencies, use the `--transitive` parameter. To show tests for a rule, use the `--include-tests` parameter. This prints out a rule's tests as if they were dependencies of the rule. To print out all of the *test's* dependencies as well, combine `--include-tests` with the `--transitive` parameter.
* `flavors <targets>`
    List the flavors that are available for the specified targets and what the default flavor is for each target. If the `flavors` command prints `no flavors`, it indicates that, although the target rule supports flavors, Buck was not able to extract any. If the `flavors` command prints `unknown`, it indicates that the target rule doesn't support flavors. The `flavors` command supports the `--json` parameter for JSON-formatted output.
* `input <targets>`
    List the input source and resource files used to build the specified targets.
* `includes <build_file>`
    List the [build file](https://buck.build/concept/build_file.html)s, and their extensions, that are included in the specified build file.
* `modules`
    List the Java modules known by Buck as well as their content hashes and dependencies.
* `ruletype <rule>`
    Print the Python signature for the specified rule.
    The following command line uses `buck audit ruletype` to view the arguments supported by the [`remote_file`](https://buck.build/rule/remote_file.html) rule.

```
buck audit ruletype remote_file
def remote_file (
    name,
    sha1,
    url,
    labels = None,
    licenses = None,
    out = None,
    type = None,
):
    ...
```

* `ruletypes`
    List all the rules that Buck supports, in alphabetical order.
    **Example**
    The following example prints *all* the rules that Buck supports. Note that the output is truncated for brevity.

```
buck audit ruletypes
android_aar
android_app_modularity
android_binary
android_build_config
android_bundle
android_instrumentation_apk
android_instrumentation_test
android_library
android_manifest
android_prebuilt_aar
android_resource
apk_genrule
apple_asset_catalog
apple_binary
<truncated>
```

**Example**
The following command line uses `buck audit ruletypes` with the `grep` command to print all the build rules that have the string `android` in their names.

```
buck audit ruletypes | grep android
```

Note that these are not all the rules that Buck provides for Android development. For example, the rules `apk_genrule` and `ndk_library` support Android development, but do not themselves contain the string `android` in their names.

* `tests <targets> [...]`
    List the tests for the specified targets. Results are listed in alphabetical order. Only tests for the specified targets are printed, though multiple targets may be specified on a single command line. This command is intended to be used in conjunction with the `audit dependencies` command. For example, to retrieve a list of all tests for a given project, use:

```
buck audit dependencies --transitive PROJECT | xargs buck audit tests
```

## Parameters

* `--include-tests`
    Show the tests for the specified targets. Can be combined with the `--transitive` parameter. For more information, see the `dependencies` command.
* `--json`
    Output the results as JSON.
* `--list`
    List `.buckconfig` and `.buckconfig.local` aliases. Used only with the `aliases` command. For more information, see that command.
* `--tab`
    Output the results using tab delimiters. Used only with the `config` command. For more information, see that command.
* `--transitive`
    Show transitive dependencies in addition to direct dependencies. Can be combined with the `--include-tests` parameter. For more information, see the `dependencies` command.

## Examples

```
## BUCK## For all of the following examples, assume this BUCK file exists in# the `examples` directory.#
java_library(
  name = 'one',
  srcs = [ '1.txt' ],
  deps = [':two',':three',],)

java_library(
  name = 'two',
  srcs = [ '2.txt' ],
  deps = [':four',],)

java_library(
  name = 'three',
  srcs = [ '3.txt' ],
  deps = [':four',':five',],)

java_library(
  name = 'four',
  srcs = [ '4.txt' ],
  deps = [':five',],)

java_library(
  name = 'five',
  srcs = [ '5.txt' ],)
```

List all of the source files used to build the `one` library

```
buck audit input //examples:one
examples/1.txt
examples/2.txt
examples/3.txt
examples/4.txt
examples/5.txt
```

Output a JSON representation of all of the source files used to build the `two` library. In this JSON object, each key is a build target and each value is an array of the source paths used to build that rule.

```
buck audit input --json //examples:two
{"//examples:two": ["examples/2.txt"],"//examples:four": ["examples/4.txt"],"//examples:five": ["examples/5.txt"],}
```

List all of the rules that the `one` library directly depends on

```
buck audit dependencies //examples:one
//examples:three
//examples:two
```

List all of the rules that the `one` library transitively depends on

```
buck audit dependencies --transitive //examples:one
//examples:five
//examples:four
//examples:three
//examples:two
```

Output a JSON representation of all of the rules that the `two` library transitively depends on.

```
buck audit dependencies --transitive --json //examples:two
{"//examples:two": ["//examples:five","//examples:four"]}
```

Output a JSON representation of the direct dependencies of the `two` and `three` libraries.

```
buck audit dependencies --json //examples:two //examples:three
{"//examples:three": ["//examples:five","//examples:four"],"//examples:two": ["//examples:four"]}
```

