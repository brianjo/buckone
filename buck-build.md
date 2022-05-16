# buck build

Builds one or more rules specified by a set of [build target](https://buck.build/concept/build_target.html)s and [build target pattern](https://buck.build/concept/build_target_pattern.html)s. This is the most commonly used command in Buck.
Syntax:

```
buck build { <build target> | <build target pattern> } . . .
```

Example:

```
buck build //java/com/example/app:amazing
```

Note that any [`remote_file`](https://buck.build/rule/remote_file.html) rules referenced by your build rules must be downloaded with [`buck fetch`](https://buck.build/command/fetch.html) prior to calling this command unless you set `in_build = true` in the `[download]` section of your [`.buckconfig`](https://buck.build/files-and-dirs/buckconfig.html).

## Parameters

* `--build-report` Specifies a file where a JSON summary of the build output should be written. Note that `--build-report` can be used without `--keep-going`, though if `--keep-going` is not specified, the generated report may be partial and not include information about the targets that buck haven't had a chance to try to build. Here is a sample build report:

```
{"success": false,"results": {"//fake:rule1": {"success": true,"type": "BUILT_LOCALLY","output": "buck-out/gen/fake/rule1.txt"},"//fake:rule2": {"success": false},"//fake:rule3": {"success": true,"type": "FETCHED_FROM_CACHE"}}}
```

In this example, both `//fake:rule1` and `//fake:rule3` were built successfully, but only `//fake:rule1` had an output file associated with the rule.
Note that this contains the same information that `--keep-going` prints to the console, but is easier to parse programmatically. This report may contain more fields in the future.

* `--keep-going` When specified, Buck will attempt to build all targets specified on the command line, even if some of the targets fail. (Buck's default behavior is to exit immediately when any of the specified targets fail.)
    When `--keep-going` is specified, a report of the build will be printed to stderr, detailing the build status of each target. Each line of the report represents the status of one build target, which has up to four columns:

    1. `OK` or `FAIL`, as per the success of the build rule.
    2. The build target of the rule.
    3. If successful, the type of the success as defined by the [`com.facebook.buck.core.build.engine.BuildRuleSuccessType`](https://buckbuild.com/javadoc/com/facebook/buck/core/build/engine/BuildRuleSuccessType.html) enum.
    4. If successful, the path to the output file of the rule, if it exists.

For example, if Buck were run with the following arguments:

```
buck build --keep-going //fake:rule1 //fake:rule2 //fake:rule3
```

Then the report printed to stderr might look like:

```
OK   //fake:rule1 BUILT_LOCALLY buck-out/gen/fake/rule1.txt
FAIL //fake:rule2
OK   //fake:rule3 FETCHED_FROM_CACHE
```

In this example, both `//fake:rule1` and `//fake:rule3` were built successfully, but only `//fake:rule1` had an output file associated with the rule. Admittedly, the state of column 1 could be derived from the presence of column 3, but the encoding of column 1 makes it easier to filter out successful rules from failed ones.
Note that when `--keep-going` is specified, the exit code will be 0 only if all targets were built successfully.
This option is analogous to `-k/--keep-going` in Make.

* `--out` Takes the output of the build target and copies it to the specified path. Only works with a single build target, and the corresponding build rule must support this option.
* `--populate-cache` Performs a cache population, which makes the output of all unchanged transitive dependencies available (if these outputs are available in the remote cache). Does not build changed or unavailable dependencies locally.
* `--show-output`
    Print the relative paths to the output for each target after the target name. Note that some rules—such as [`genrule`](https://buck.build/rule/genrule.html)—generate sources instead of build artifacts.

#### Additional Parameters

For additional parameters that are available with this command, see the [Common Parameters](https://buck.build/command/common_parameters.html) topic.

## Examples

### View compiler flags using the compilation database

For C++ builds, to view the compiler flags that Buck is passing to the C++ compiler, tell Buck to generate a *compilation database* for the target. The compilation database is a JSON file that contains information about how Buck compiled the target. The syntax is:

```
buck build path/to/target#compilation-database
```

You can then get the path to the compilation-database file itself using:

```
buck targets --show-output path/to/target#compilation-database
```

For example:

```
$ buck cleanParsing buck files: finished in 1.7 sec (100%)Building: finished in 1.5 sec (100%) 3/3 jobs, 3 updated, 0.0% cache miss
  Total time: 3.7 sec

$ buck build :main#compilation-databaseBuilding: finished in 1.5 sec (100%) 3/3 jobs, 3 updated, 0.0% cache miss
  Total time: 3.7 sec

$ buck targets --show-output :main#compilation-database//:main#compilation-database buck-out/gen/__main#compilation-database/compile_commands.json

$ jq . buck-out/gen/__main#compilation-database/compile_commands.json [{"directory": "/Users/devuser/git/gtDev/C++/oop/simple-classes","file": "/Users/devuser/git/gtDev/C++/oop/simple-classes/main.cpp","arguments": ["/usr/bin/clang++","-x","c++","-I","buck-out/gen/main#default,private-headers.hmap","-I","buck-out","-Xclang","-fdebug-compilation-dir","-Xclang",".","-fdebug-prefix-map=/Users/devuser/git/gtDev/C++/oop/simple-classes=.","-c","-MD","-MF","buck-out/gen/main#compile-main.cpp.oa5b6a1ba,default/main.cpp.o.dep","main.cpp","-o","buck-out/gen/main#compile-main.cpp.oa5b6a1ba,default/main.cpp.o"]}]
```

In the preceding example `:main` specifies a *relative* build target that is defined in the build file located in the current directory. The `jq` tool—available through package managers such as Homebrew—enables you to pretty-print JSON files.
