# Commands

# Common Parameters

This topic describes command-line parameters that affect the operation of Buck itself and that are available irrespective of which subcommand (`build`, `run`,`test`, etc) you invoke.

## Parameters

* `--verbose`
    Set the verbosity level of the console output. The verbosity level is a single numeric value between `1` and `8` inclusive. For example:

```
buck targets --verbose 8
```

Higher verbosity levels include all the output of lower levels.
The output that Buck produces is affected by factors in addition to the verbosity level. Such factors include the subcommand (`build`, `install`, `test`, etc), the types of rules being built, and the degree to which artifacts from previous builds have been cached.
Buck has not yet implemented differences between all levels. For example, there are currently no differences in verbosity between levels `5` and `7` inclusive.

|Numeric level	|Boolean function	|Description	|
|---	|---	|---	|
|1	|`shouldPrintStandardInformation()`	|Print warnings from build steps and summary information for tests.	|
|---	|---	|---	|
|2	|`shouldPrintBinaryRunInformation()`	|Print additional output for generated binaries or tests.	|
|3	|`shouldPrintCommand()`	|Print commands that Buck runs during the build process.	|
|4	|`shouldPrintSelectCommandOutput()`	|Print output for selected commands that Buck runs.	|
|`5 - 7`	|`shouldPrintOutput()`	|Print output for all commands that Buck runs.	|
|8	|`shouldUseVerbosityFlagIfAvailable()`	|Print all available diagnostic/logging information.	|

For more precise information about how a particular verbosity level affects output, you can search the Buck source code for the corresponding boolean function. For example:

```
grep --recursive "getVerbosity().shouldPrintOutput()" ~/local/buck/src
```

* `--no-cache`
    Disable the build artifact caches. If this parameter is specified, Buck ignores any artifacts in any of the caches specified in the [`[cache]`](https://buck.build/files-and-dirs/buckconfig.html#cache) section of `.buckconfig`. These include the filesystem cache (`dir`), remote cache (`http`), and SQLite cache (`sqlite`).
    The contents of the caches are unaffected, but Buck does not use them for the specified command.
    Note that if there is an output file in the `buck-out` directory for a previously-built and unchanged rule, Buck still uses that output file in your build—even if `--no-cache` is specified. If you don't want Buck to use these (valid) build artifacts, run the [`buck clean`](https://buck.build/command/clean.html) command before your build to delete them from `buck-out`.
* `--config`
    Specify configuration settings or override existing settings in [`.buckconfig`](https://buck.build/files-and-dirs/buckconfig.html). For example:

```
buck build --config cache.mode=dir //java/com/example/app:amazing
```

The `--config` parameter can be abbreviated as `-c`.
You can specify the `--config` parameter multiple times on the same command line. Note, however, that if the same configuration option is specified more than once, Buck uses the last value specified ("last write wins"). For example, the following invocation of `buck build` builds the `//:prod` target, rather than the `//:dev` target, which was specified earlier on the command line. (The example uses the abbreviated, `-c`, version of the parameter.)

```
#
# Build for development? 
#
# No, build for production.
#
buck build -c 'alias.main=//:dev' -c 'alias.main=//:prod' main
```

The preferred method of overriding values in `.buckconfig` is by using a `.buckconfig.local` file. Overriding values of `.buckconfig` from the command line can make it difficult to reproduce builds.

* `--config-file`
    Specify build-configuration settings in a file that uses the same syntax as [`.buckconfig`](https://buck.build/files-and-dirs/buckconfig.html). For example:

```
buck build --config-file debug.buckconfig
```

The `--config-file` parameter provides functionality similar to `--flagfile` (see below), but with `.buckconfig` syntax rather than command-line parameter syntax.
Any values specified using `--config-file` override values specified in `.buckconfig` and `.buckconfig.local`.
You can specify the path to the configuration file in one of three ways.

##### Use a path that is relative to the directory that contains the current cell's** **`.buckconfig`.

```
--config-file relative/path/to/file.buckconfig
```

##### Use a path that is relative to the directory that contains a** ***specified*** **cell's** **`.buckconfig`.

```
--config-file <cell name>//path/to/file.buckconfig
```

##### Use an absolute path from the root of your file system.

```
--config-file /absolute/path/to/file.buckconfig
```

You can also specify a particular cell to which to apply the configuration. By default, the settings in the configuration file apply to *all* cells in the current build.

##### Apply the configuration only to the** ***current*** **cell.

```
--config-file //=<path-to-config-file>
```

##### Apply the configuration only to a** ***specified*** **cell.

```
--config-file <target-cell>=<path-to-config-file>
```

If you specify `*` as the target cell, the configuration is applied to *all* the cells in the build. This is the default, but this syntax enables you to be explicit.
To learn more about Buck's concept of cells, see the [Key Concepts](https://buck.build/about/overview.html) topic.

* `--num-threads`
    Specify the number of threads that buck uses when executing jobs. The default value is 1.25 times the number of *logical* cores in the system; on systems with hyperthreading, this means that each physical core is counted as two logical cores. You can also set the number of threads to use for building by adding a `threads` key to the `[build]` section of the `.buckconfig` file.
    The order of precedence for setting the number of build threads (from highest to lowest) is:

    1. The `--num-threads` command-line parameter.
    2. The `.buckconfig` setting.
    3. The default value (see above).

The number of *active* threads might not always be equal to this argument.

* `--flagfile /path/to/commandline-args or @/path/to/commandline-args`
    Specify additional command-line arguments in external files (*flag files*), one argument per line. The arguments in these files can themselves be `--flagfile` or `@` arguments, which would then include the additional specified file's contents as arguments.

```
# File config/common
--verbose
# File config/gcc
@config/common
--config
cxx.cxx=/usr/bin/g++
...
# File config/clang
@config/common
--config
cxx.cxx=/usr/bin/clang++
...
buck build @config/gcc foo/bar:
buck build @config/clang foo/bar:
```

Lines in flag files must not have any leading or trailing white space.
The equals sign (`=`) separates the specified property and value. There should be no white space between the property and the equals sign, nor between the equals sign and the value.
We recommend that you use `--flagfile` rather than the `@` symbol as `--flagfile` is more self-describing.
This `--config-file` parameter (described earlier) provides functionality similar to `--flagfile`, but with `.buckconfig` syntax rather than command-line parameter syntax.
If Buck is regularly invoked with different sets of arguments, we recommend that you use flag files or config files, as they can be stored in source control, which makes builds more reproducible.



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



# buck clean

Delete artifacts and state files generated by Buck.
Buck generates artifacts in the form of files and Buck also maintains its state using files. Deleting these files resets Buck's state so that the next build that runs is guaranteed to be a *clean build*.
The set of files that Buck generates and the location where Buck stores these files change over time because these are a function of the Build configuration and the version of Buck that you are using. However, the `buck clean` command always removes the appropriate set of files for your build and Buck version.

## Do not manually try to modify the buck-out directory

Buck primarily stores the state of your build in the `buck-out` directory which is located in the root of your [Buck project](https://buck.build/about/overview.html). However, you should not try to manually modify the contents of `buck-out`. Buck's implementation assumes that `buck-out` is not touched by anything external to Buck. If you modify `buck-out`, Buck might behave in ways that are unpredictable.

## Parameters

* `--dry-run`
    Print a list of the directories that would be removed if the command ran normally.
* `--keep-cache`
    Keep the local (dir) caches. For more information about the local caches, see the [`[cache]`](https://buck.build/files-and-dirs/buckconfig.html#cache) section of `.buckconfig`.

#### Additional Parameters

For additional parameters that are available with this command, see the [Common Parameters](https://buck.build/command/common_parameters.html) topic.


# buck doctor

The purpose of this command is to help the user debug and solve and issue of another buck command. This command requires implementing custom server-side support to be fully functional. We do not provide a functional implementation with buck, however we have documented the protocol and the`DoctorCommandIntegrationTest` contains very simple examples of interactions between the doctor command and server.
You can use this command after setting up the environment.

```
buck doctor
```

## Command flow

The command supports the debugging of other commands except invocations of `buck rage`and `buck doctor`. At the start the user has to choose the buck invocation that they want to debug. In order for a buck invocation to appear on the list it must have log files in `buck-out`. After choosing the command a report is generated and either stored locally or is uploaded to a remote endpoint defined under `[rage]` in buckconfig. Also, another Json report is generated structured after class `AbstractDoctorEndpointRequest` and is send to the remote endpoint defined under `[doctor]` in buckconfig. This report also contains a URL to the location of the rage report. Then, the doctor remote endpoint produces a response that contains information about the status of core components like parsing, environment and remote cache along with suggestions on how to solve potential problems.

## Setup notes

* The command requires the creation of at least one remote endpoint that handles the Json request and creates the fixing suggestions based on the Json format of`AbstractDoctorEndpointResponse`.
* If you want to also have the data from the creation of the rage report you will need to define another remote endpoint that can get the information, otherwise the zipped report will be saved on disk
* All the communication is sent using POST.

## Formats & Structures

* Request is based on the Json format of class `AbstractDoctorEndpointRequest`.
* Response is based on the Json format of class `AbstractDoctorEndpointResponse`.
* The rage report zip file contains a Json file will metadata and useful information along with a list of log files and local configuration.

You can pass extra arguments to the doctor request by defining the `[doctor_extra_request_args]`. This will add them as extra parameters in the post request. In this case the main information will be under parameter `data`.

## Buckconfig example

```
[rage]
    report_upload_url = https://localhost:4546
    report_max_size = 100MB

[doctor]
    endpoint_url = https://localhost:4545
    extra_request_args = ref=>12ab,token=>42
```

## Request example

```
{
    buildId: "...",
    rageUrl: "...",
    machineReadableLog: "...",
}
```

## Response example

```
{
    errorMessage: "...",
    suggestions: [
                   "state": "ERROR",
                   "area": "Environment",
                   "suggestion": "Parsing too slow. Check BUCK file.",
                 ]
}


```




# buck doctor

The purpose of this command is to help the user debug and solve and issue of another buck command. This command requires implementing custom server-side support to be fully functional. We do not provide a functional implementation with buck, however we have documented the protocol and the`DoctorCommandIntegrationTest` contains very simple examples of interactions between the doctor command and server.
You can use this command after setting up the environment.

```
buck doctor
```

## Command flow

The command supports the debugging of other commands except invocations of `buck rage`and `buck doctor`. At the start the user has to choose the buck invocation that they want to debug. In order for a buck invocation to appear on the list it must have log files in `buck-out`. After choosing the command a report is generated and either stored locally or is uploaded to a remote endpoint defined under `[rage]` in buckconfig. Also, another Json report is generated structured after class `AbstractDoctorEndpointRequest` and is send to the remote endpoint defined under `[doctor]` in buckconfig. This report also contains a URL to the location of the rage report. Then, the doctor remote endpoint produces a response that contains information about the status of core components like parsing, environment and remote cache along with suggestions on how to solve potential problems.

## Setup notes

* The command requires the creation of at least one remote endpoint that handles the Json request and creates the fixing suggestions based on the Json format of`AbstractDoctorEndpointResponse`.
* If you want to also have the data from the creation of the rage report you will need to define another remote endpoint that can get the information, otherwise the zipped report will be saved on disk
* All the communication is sent using POST.

## Formats & Structures

* Request is based on the Json format of class `AbstractDoctorEndpointRequest`.
* Response is based on the Json format of class `AbstractDoctorEndpointResponse`.
* The rage report zip file contains a Json file will metadata and useful information along with a list of log files and local configuration.

You can pass extra arguments to the doctor request by defining the `[doctor_extra_request_args]`. This will add them as extra parameters in the post request. In this case the main information will be under parameter `data`.

## Buckconfig example

```
[rage]
    report_upload_url = https://localhost:4546
    report_max_size = 100MB

[doctor]
    endpoint_url = https://localhost:4545
    extra_request_args = ref=>12ab,token=>42
```

## Request example

```
{
    buildId: "...",
    rageUrl: "...",
    machineReadableLog: "...",
}
```

## Response example

```
{
    errorMessage: "...",
    suggestions: [
                   "state": "ERROR",
                   "area": "Environment",
                   "suggestion": "Parsing too slow. Check BUCK file.",
                 ]
}


```



# buck fix

Attempts to fix errors and warnings encountered during the previous command.
Commands such as [`buck build`](https://buck.build/command/build.html) may fail with errors from Buck itself or another tool (such as a compiler or linker) that is invoked by Buck. In some cases, such errors can be fixed automatically. In such cases, running

```
buck fix
```

may fix the error.
`buck fix` prints nothing to the console unless a catastrophic failure occurs.
Note that `buck fix` will edit files in place. It is therefore recommended that any pending changes be saved to source control before running this command.




# buck install

This command builds and installs an `.apk` or `.app` bundle on a emulator/simulator or device, and optionally launches it.

## Common Parameters

All the parameters for [`buck build`](https://buck.build/command/build.html) also apply to `buck install`.

* `--run` `(-r)` Launch the `.apk` with the default activity (Android) or the `.app` bundle (iOS) after installation.
* `--emulator` `(-e)` Use this option to use emulators/simulators only.
* `--device` `(-d)` Use this option to use real devices only. This option works only with Android devices. It does not work with iOS devices.
* `--serial` `(--udid)` Use device or emulator/simulator with specific serial or UDID number.

## Android

Builds and installs the APK for an `android_binary` or target.
Takes an [`android_binary`](https://buck.build/rule/android_binary.html), an [`apk_genrule`](https://buck.build/rule/apk_genrule.html) or an [`android_instrumentation_apk`](https://buck.build/rule/android_instrumentation_apk.html), builds it, and installs it by running `adb install <path_to_the_APK>`.

### Parameters

* `--activity <fully qualified class name>` `(-a)` Launch the `.apk` with the specified activity after installation.
* `-all` `(-x)` Install APK on all connected devices and/or emulators (multi-install mode).
* `--adb-threads` `(-T)` Number of threads to use for adb operations. Defaults to number of connected devices.

## iOS

Builds and installs an .app for an `apple_bundle` target.
Takes an [`apple_bundle`](https://buck.build/rule/apple_bundle.html), builds it, and installs it by copying it to a simulator or device as appropriate.
For device support, you need to first build the `fbsimctl` utility from [FBSimulatorControl](https://github.com/facebook/FBSimulatorControl/) and set [`[apple].device_helper_path`](https://buck.build/files-and-dirs/buckconfig.html#apple.device_helper_path) to its location.

### Parameters

* `--simulator-name` `(-n)` Use simulator with specific name (defaults to `iPhone 8`)



# buck kill

Kill the [Buck Daemon (`buckd`)](https://buck.build/concept/buckd.html) for the current project. To kill all the Buck Daemon processes running on the host computer, use [`buck killall`](https://buck.build/command/killall.html).
For an explanation of Buck's concept of a *project*, see the the [Key Concepts](https://buck.build/about/overview.html) topic.


# buck killall

Kill *all* the [Buck Daemon (`buckd`)](https://buck.build/concept/buckd.html) processes that are running on the host computer.
To kill only the Buck Daemon process for the *current* Buck [project](https://buck.build/about/overview.html), use [`buck kill`](https://buck.build/command/kill.html).
In general, we recommend using `buck kill` rather than `buck killall`. The `buck killall` command is an aggressive way of terminating the `buckd` processes and might leave your build environment in an inconsistent state.



# buck project

This command generates the configuration files for an IDE to work with the project. This command creates files in-place in the repository, which is unlike other Buck commands whose output is removed by [`buck clean`](https://buck.build/command/clean.html). As a result, it is a good idea to add these generated files to the list of ignored files by your choice of source control. IDE-specific details are discussed in each section below.
You can use this command by itself to generate a project for the entire repository.

```
buck project
```

You can also use this command to build a project slice (a project that represents a subset of the repository). You can pass any number of [build target](https://buck.build/concept/build_target.html)s or [build target pattern](https://buck.build/concept/build_target_pattern.html)s to the command. The constructed project slice will contain the specified targets and their dependencies. This is useful for large repositories.

```
buck project //java/...
```

## Common Parameters

* `--ide` Specifies which IDE to create the project for. When using a project slice, Buck tries to determine what type of IDE to use automatically based on the [build target](https://buck.build/concept/build_target.html)s provided. Sometimes it is not possible to determine the type of IDE. You can specify the default ide in the `[project]` section of your `.buckconfig` file.
* `--without-tests` Indicates that Buck should build a project slice without tests (the default is to include `tests` on `*_library` and `*_binary` rules).
* `--without-dependencies-tests` Indicates that Buck should build a project slice with the tests of the specified targets only.
* `--exclude-artifacts` ``Don't include references to the artifacts created by compiling a target in the module representing that target. This can improve indexing times, but will mean generated code does not show up in the ide. For example R files for Android.
* `--remove-unused-ij-libraries` ``After generating an IntelliJ project remove all IntelliJ libraries that are not used in the project.

## Supported IDEs

* [IntelliJ](https://buck.build/command/project.html#intellij)
* [Xcode](https://buck.build/command/project.html#xcode)
* gobuild (Go IDEs such as Visual Sudio Code, GoLand)

### IntelliJ

This command processes all of the [build file](https://buck.build/concept/build_file.html)s whose targets were specified and uses them to generate the configuration files for an IDE. The generated files include:

* `.idea/libraries/*.xml`, each of which defines a library in IntelliJ. A library always corresponds to a [`prebuilt_jar`](https://buck.build/rule/prebuilt_jar.html).
* `.iml` files, each of which defines a module in IntelliJ. A module can depend on other modules, as well as libraries. It should be noted that although Buck allows multiple build targets per build file, IntelliJ's modules are only defined at the directory level. This means that you may find IntelliJ flagging compilation errors because of missing dependencies of classes outside of your project slice, but which happen to be in the same directory as classes within the slice.
* `.idea/modules.xml`, which lists all of the IntelliJ modules in the project.

### Xcode

This command processes each [`apple_binary`](https://buck.build/rule/apple_binary.html), [`apple_bundle`](https://buck.build/rule/apple_bundle.html), and [`apple_library`](https://buck.build/rule/apple_library.html) specified, and uses them to generate the files and directories that Xcode needs. The generated folders include:

* For each [build target](https://buck.build/concept/build_target.html), an `*.xcworkspace` directory that represents the [workspace](https://developer.apple.com/library/ios/featuredarticles/XcodeConcepts/Concept-Workspace.html) and contains one or more [schemes](https://developer.apple.com/library/ios/featuredarticles/XcodeConcepts/Concept-Schemes.html).
* For each [build target](https://buck.build/concept/build_target.html) and its dependencies, an `*.xcodeproj` directory that represents the [project](https://developer.apple.com/library/ios/featuredarticles/XcodeConcepts/Concept-Projects.html). These generated projects are only buildable within the generated workspace.

#### Parameters

* `--combined-project` Indicates that Buck should build a single monoproject for all [build target](https://buck.build/concept/build_target.html)s specified.
* `--project-schemes` Enables the generation of separate schemes for all projects included in the generated workspace. Each project's scheme is constrained to the set of build and test targets that are members or deps of the associated project. You can specify the default value using by adding `project_schemes = true` to the`[project]` section of your `.buckconfig` file.

### Go IDEs

Go IDEs, such as Visual Studio Code and GoLand, use the `go build` command to build Go code. This command requires a proper layout of source code under `GOPATH`. However, Buck puts generated code in `buck-out`, which `go build` does not understand. The `project` command copies generated Go packages from `buck-out` and arranges them in the same layout as their import paths.

#### Parameters

* `--project_path` The directory to copy the generated code to. If not specified, the `project` command copies it to `\/\/vendor`, or `\/\/src/vendor` if it exists.



# buck publish

A command that builds one or more [build target](https://buck.build/concept/build_target.html)s and publishes them. This does not support [build target pattern](https://buck.build/concept/build_target_pattern.html)s.

## Parameters

All parameters that can be passed to [`buck build`](https://buck.build/command/build.html) can be passed to this command to control how the [build target](https://buck.build/concept/build_target.html)s are built. The following parameters are also supported:

* `--dry-run`
    Just print the artifact(s) that would be published for the specified [build target](https://buck.build/concept/build_target.html)(s).
* `--include-source` `(-s)`
    Publish the source code in addition to the artifact(s) for the specified [build target](https://buck.build/concept/build_target.html)(s).
* `--remote-repo` `(-r)`
    A url of the remote repository to publish artifact(s) for the specified [build target](https://buck.build/concept/build_target.html)(s).
* `--username` `(-u)`
    The username to use when authenticating with the remote repository.
* `--password` `(-p)`
    The password to use when authenticating with the remote repository.
* `--to-maven-central`
    Equivalent to passing `--remote-repo https://repo1.maven.org/maven2`.



# buck query

The `buck query` command provides functionality to query the *target-nodes* graph ("target graph") and return the build targets that satisfy the query expression.
The query language enables you to combine multiple operators in a single command. For example, to retrieve a list of all the tests for a build target, you can combine the `deps()` and `testsof()` operators into a single call to `buck query`.

```
buck query "testsof(deps('//java/com/example/app:amazing'))"
```

## Query Language

The Buck query language was inspired by the [Bazel Query Language](http://bazel.io/docs/query.html). Buck's query language uses the same parser, so the lexical syntax is similar. Buck's query language supports a *subset* of Bazel's query functionality but also adds a few operators, such as `attrfilter`, `inputs`, and `owner`.

### Operators

Buck's query language supports the following operators. The name of the operator below is linked to a section that provides more detail about that operator's functionality and syntax.

* [`allpaths`](https://buck.build/command/query.html#allpaths): All dependency paths
* [`attrfilter`](https://buck.build/command/query.html#attrfilter): Rule attribute filtering
* [`attrregexfilter`](https://buck.build/command/query.html#attrregexfilter): Rule attribute filtering with regex
* [`buildfile`](https://buck.build/command/query.html#buildfile): Build files of targets
* [`deps and first-order-deps`](https://buck.build/command/query.html#deps): Transitive closure of dependencies
* [`except`](https://buck.build/command/query.html#set-operations): Set difference
* [`filter`](https://buck.build/command/query.html#filter): Filter targets by name
* [`inputs`](https://buck.build/command/query.html#inputs): Direct input files
* [`intersect`](https://buck.build/command/query.html#set-operations): Set intersection
* [`kind`](https://buck.build/command/query.html#kind): Filter targets by rule type
* [`labels`](https://buck.build/command/query.html#labels): Extract content of rule attributes
* [`owner`](https://buck.build/command/query.html#owner): Find targets that own specified files
* [`rdeps`](https://buck.build/command/query.html#rdeps): Transitive closure of reverse dependencies
* [`set`](https://buck.build/command/query.html#set): Group targets
* [`testsof`](https://buck.build/command/query.html#testsof): List the tests of the specified targets
* [`union`](https://buck.build/command/query.html#set-operations): Set union

### Parameters to operators

The most common parameter for a Buck query operator is an expression that evaluates to a build target or collection of build targets. Such an expression could be an explicit [build target](https://buck.build/concept/build_target.html), a [build target pattern](https://buck.build/concept/build_target_pattern.html), an [`[alias]`](https://buck.build/files-and-dirs/buckconfig.html#alias), or *the set of targets returned by another Buck query operator*.
**Tip:** You can pass an alias directly to the `buck query` command line to see what it resolves to. For example:

```
$ buck query app
//apps/myapp:app
```

#### Non-target parameters

In addition to target parameters, some Buck query operators take string parameters such as filenames ([`owner()`](https://buck.build/command/query.html#owner)) or regular expressions ([`filter()`](https://buck.build/command/query.html#filter)).
**Note:** You can hover over the parameters in the syntax blocks for the query operators (later in this topic) to obtain short tool-tip descriptions of the parameters.

#### Quoting of arguments

It is not necessary to quote arguments if they adhere to certain conditions. That is, they comprise sequences of characters drawn from the alphabet, numerals, forward slash (`/`), colon (`:`), period (`.`), hyphen (`-`), underscore (`_`), or asterisk (`*`)—and they do not start with a hyphen or period. For example, quoting `java_test` is unnecessary.
All that said, we *do* recommend that you quote arguments as a best practice even when Buck doesn't require it.
You should always use quotes when writing scripts that construct `buck query` expressions *from user-supplied values*.
Note that argument quoting for `buck query` is in addition to any quoting that your shell requires. In the following example, double-quotes are used for the shell and single-quotes are used for the `build target` expression.

```
buck query "'//foo:bar=wiz'"
```

#### Algebraic set operations: intersection, union, set difference

|Nominal	|Symbolic	|
|---	|---	|
|`intersect`	|`^`	|
|---	|---	|
|`union`	|`+`	|
|`except`	|`-`	|

These three operators compute the corresponding set operations over their arguments. Each operator has two forms, a nominal form, such as `intersect`, and a symbolic form, such as `^`. the two forms are equivalent; the symbolic forms are just faster to type. For example,

```
buck query "deps('//foo:bar') intersect deps('//baz:lib')"
```

and

```
buck query "deps('//foo:bar') ^ deps('//baz:lib')"
```

both return the targets that appear in the [transitive closure](https://en.wikipedia.org/wiki/Transitive_closure) of `//foo:bar` and `//baz:lib`.
The `intersect` (`^`) and `union` (`+`) operators are commutative. The `except` (`-`) operator is not commutative.
The parser treats all three operators as left-associative and of equal precedence, so we recommend that you use parentheses if you need to ensure a specific order of evaluation. A parenthesized expression resolves to the value of the expression it encloses. For example, the first two expressions are equivalent, but the third is not:

```
x intersect y union z
(x intersect y) union z
x intersect (y union z)
```

#### Group targets: set

**Syntax**

```
set(**a:expr** *b:expr* *c:expr* ...)
```

The `set(`*`a`*` `*`b`*` `*`c`*` ...)` operator computes the union of a set of zero or more target expressions. Separate the targets with white space (not commas). Quote the targets to ensure that they are parsed correctly.
If you want to invoke `buck query` on a list of targets, then `set()` is a way to group this list in a query.
**Example:**
The following command line returns the target `main` in the build file in the root of the Buck project and all the targets from the build file in the `myclass` subdirectory of the root.

```
buck query "set( '//:main' '//myclass:' )"
```

**Example:**
The following command line returns the merged set (union) of dependencies for the targets: `main` and `subs` in the build file in the root of the Buck project.

```
buck query "deps( set( '//:main' '//:subs' ) )"
```

#### All dependency paths: allpaths

**Syntax**

```
allpaths(*from:expr*, *to:expr*)
```

The `allpaths(`*`from`*`, `*`to`*`)` operator evaluates to the graph formed by paths between the target expressions *`from`* and *`to`*, following the dependencies between nodes. For example, the value of

```
buck query "allpaths('//foo:bar', '//foo/bar/lib:baz')"
```

is the dependency graph rooted at the single target node `//foo:bar`, that includes all target nodes that depend on `//foo/bar/lib:baz`.
The two arguments to `allpaths()` can themselves be expressions. For example, the command:

```
buck query "allpaths(kind(java_library, '//...'), '//foo:bar')"
```

shows all the paths between any `java_library` in the repository and the target `//foo:bar`.
We recommend using `allpaths()` with the `--output-format dot` parameter to generate a graphviz file that can then be rendered as an image. For example:

```
$ buck query "allpaths('//foo:bar', '//foo/bar/lib:baz')" --output-format dot --output-file result.dot
$ dot -Tpng result.dot -o image.png
```

*Graphviz* is an open-source graph-visualization software tool. Graphviz uses the *dot* language to describe graphs.

#### Rule attribute filtering: attrfilter

**Syntax**

```
attrfilter(*attribute*, *value*, *expr*)
```

The `attrfilter(`*`attribute`*`, `*`value`*`, `*`expr`*`)` operator evaluates the given target expression and filters the resulting build targets to those where the specified *`attribute`* contains the specified *`value`*.
In this context, the term *`attribute`* refers to an argument in a build rule, such as `name`, `headers`, `srcs`, or `deps`.
If the attribute is a single value, say `name`, it is compared to the specified *`value`*, and the target is returned if they match. If the attribute is a list, the target is returned if that list contains the specified *`value`*. If the attribute is a dictionary, the target is returned if the *`value`* exists in either the keys or the values of the dictionary.
For example,

```
buck query "attrfilter(deps, '//foo:bar', '//...')"
```

returns the build targets in the repository that depend on `//foo:bar`—or more precisely: those build targets that include `//foo:bar` in their `deps` argument list.
The match performed by `attrfilter()` is semantic rather than textual. So, for example, if you have the following `deps` argument in your build file:

```
cxx_binary(
  name = 'main',
  srcs = [
    'main.cpp'
  ],
  deps = [
    ':myclass',
  ],
)
```

Your `attrfilter()` clause should be:

```
buck query "attrfilter( deps, '//:myclass', '//...' )"
```

Note the double forward slash (`//`) before the second argument to `attrfilter()`.

#### Rule attribute filtering with regex: attrregexfilter

**Syntax**

```
attrregexfilter(*attribute*, *pattern*, *expr*)
```

The `attrregexfilter(`*`attribute`*`, `*`pattern`*`, `*`expr`*`)` operator is identical to the `attrfilter(`*`attribute`*`, `*`value`*`, `*`expr`*`)` operator except that it takes a regular expression as the second argument. It evaluates the given target expression and filters the resulting build targets to those where the specified *`attribute`* matches the specified *`pattern`*.
In this context, the term *`attribute`* refers to an argument in a build rule, such as `name`, `headers`, `srcs`, or `deps`.
If the attribute is a single value, say `name`, it is matched against the specified *`pattern`*, and the target is returned if they match. If the attribute is a list, the target is returned if that list contains a value that matches the specified *`pattern`*. If the attribute is a dictionary, the target is returned if the *`pattern`* match is found in either the keys or the values of the dictionary.

#### Build files of targets: buildfile

**Syntax**

```
buildfile(*expr*)
```

The `buildfile(`*`expr`*`)` operator evaluates to those build files that define the targets that result from the evaluation of the target expression, *`expr`*.
In order to find the build file associated with a source file, combine the `owner` operator with `buildfile`. For example,

```
buck query "buildfile(owner('foo/bar/main.cpp'))" 
```

first finds the targets that *own* `foo/bar/main.cpp` and then returns the build files, such as `foo/bar/BUCK`, that define those targets.

#### Transitive closure of dependencies: deps and first-order-deps

**Syntax**

```
deps(*argset:expr*)
deps(*argset:expr*, *depth:int*)
deps(*argset:expr*, *depth:int*, *filter:expr*)
deps(*argset:expr*, *depth:int*, first_order_deps())
```

The `deps(`*`x`*`)` operator evaluates to the graph formed by the [transitive closure](https://en.wikipedia.org/wiki/Transitive_closure) of the dependencies of its argument set, *x*, including the nodes from the argument set itself. For example, the value of

```
buck query "deps('//foo:bar')"
```

is the dependency graph rooted at the target node `//foo:bar`. It includes all of the dependencies of `//foo:bar`. It also includes `//foo:bar` itself.
The `deps` operator accepts an optional second argument, which is an integer literal specifying an upper bound on the depth of the search. So,

```
deps('//foo:bar', 1)
```

evaluates to the direct dependencies of the target `//foo:bar`, and

```
deps('//foo:bar', 2)
```

further includes the nodes directly reachable from the nodes in `deps('//foo:bar', 1)`, and so on. If the depth parameter is omitted, the search is unbounded, that is, it computes the entire transitive closure of dependencies.

#### Filter expressions and first_order_deps()

The `deps()` operator also accepts an optional third argument, which is a filter expression that is evaluated for each node and returns the child nodes to recurse on when collecting transitive dependencies.
This filter expression can use the `first_order_deps()` operator which returns a set that contains the first-order dependencies of the current node—which is equivalent to `deps(<node>, 1)`. For example, the query,

```
buck query "deps('//foo:bar', 1, first_order_deps())"
```

is equivalent to

```
buck query "deps('//foo:bar', 1)"
```

The `first_order_deps()` operator can be used only as an argument passed to `deps()`.
Note that because `deps()` uses positional parameters, you must specify the second argument in order to specify the third. In this scenario, if you want the search to be unbounded, we recommend that you use `2147483647` which corresponds to Java's `Integer.MAX_VALUE`.

#### Filter targets by name: filter

**Syntax**

```
filter(*regex*, *expr*)
```

The `filter(`*`regex`*`, `*`expr`*`)` operator evaluates the specified target expression, *`expr`*, and returns the targets that have a name attribute that matches the specified regular expression *`regex`*. For example,

```
buck query "filter('library', deps('//foo:bar'))"
```

returns the targets in the transitive closure of `//foo:bar`that contain the string `library` in their name attribute.
The `filter()` operator performs a *partial* match. So, both of the following clauses would match a target with the name `main`.

```
buck query "filter( 'main', '//...' )"
buck query "filter( 'mai', '//...' )"
```

Another example:

```
buck query "filter('.*\.java$', labels(srcs, '//foo:bar'))"
```

returns the `java` files used to build `//foo:bar`.
You often need to quote the pattern to ensure that regular expressions, such as `.*xpto`, are parsed correctly.

#### Direct input files: inputs

**Syntax**

```
inputs(*expr*)
```

The `inputs(`*`expr`*`)` operator returns the files that are inputs to the target expression, *`expr`*, ignoring all dependencies. Note that it does not include any files required for parsing, such as the BUCK file. Rather, it returns only the files required to actually run the build after parsing has been performed.
Note also that `inputs()` returns only those input files indicated by the *target graph*. Input files that are present in the *action graph* but not in the target graph are not returned by `inputs()`.
You could consider the `inputs()` and `owner()` operators to be inverses of each other.

#### Filter targets by rule type: kind

**Syntax**

```
kind(*regex*, *expr*)
```

The `kind(`*`regex`*`, `*`expr`*`)` operator evaluates the specified target expression, *`expr`*, and returns the targets where the rule type matches the specified *`regex`*. For example,

```
buck query "kind('java_library', deps('//foo:bar'))"
```

returns all `java_library` targets in the transitive dependencies of `//foo:bar`.
The specified *`pattern`* can be a regular expression. For example,

```
buck query "kind('.*_test', '//...')"
```

returns all targets in the repository with a rule type that ends with `_test`, such as `java_test` and `cxx_test`.
You often need to quote the pattern to ensure that regular expressions, such as `.*xpto`, are parsed correctly.
To get a list of the available rule types in a given set of targets, you could use a command such as the following:

```
buck query : --output-attribute buck.type
```

which prints all the rule types in the build file in the current directory (`:`)—in JSON format. See `--output-attribute` described in the **Parameters** section below for more information.

#### Extract content of rule attributes: labels

**Syntax**

```
labels(*attribute*, *expr*)
```

The `labels(`*`attribute`*`, `*`expr`*`)` operator returns the set of build targets and file paths listed in the attribute specified by the *`attribute`* parameter, in the targets that result from the evaluation of target expression, *`expr`*. Valid values for *attribute* include `srcs`, `headers`, and `deps`.
**Example**: Get all build targets and file paths specified in the `srcs` attribute for *all the rules* in the build file in the current directory.

```
buck query "labels( 'srcs', ':' )"
```

In performing this operation, Buck validates that any source files referenced in these attributes do, in fact, exist; Buck generates an error if they do not.
**Example**: Get all the build targets and file paths specified in the `deps` arguments in the *tests of* the target `//foo:bar`.

```
buck query "labels('deps', testsof('//foo:bar'))"
```

Note that `deps` must be quoted because, in addition to being a build-file attribute, it is itself a reserved keyword of the query language.

#### Find targets that own specified files: owner

**Syntax**

```
owner(*inputfile*)
```

The `owner(`*`inputfile`*`)` operator returns the targets that own the specified *`inputfile`*. In this context, *own* means that the target has the specified file as an input. You could consider the `owner()` and `inputs()` operators to be inverses of each other.
**Example**:

```
buck query "owner('examples/1.txt')"
```

returns the targets that owns the file `examples/1.txt`, which could be a value such as `//examples:one`.
It is possible for the specified file to have multiple owners, in which case, `owner()` returns a set of targets.
If no owner for the file is found, `owner()` outputs the message:

```
No owner was found for <file>
```

#### Transitive closure of reverse dependencies: rdeps

**Syntax**

```
rdeps(*universe:expr*, *argset:expr*)
rdeps(*universe:expr*, *argset:expr*, *depth:int*)
```

The `rdeps(universe, argset)` operator returns the reverse dependencies of the argument set *`argset`* within the [transitive closure](https://en.wikipedia.org/wiki/Transitive_closure) of the set *`universe`* (the *universe*). The returned values include the nodes from the argument set *`argset`* itself.
The `rdeps` operator accepts an optional third argument, which is an integer literal specifying an upper bound on the depth of the search. A value of one (`1`) specifies that `buck query` should return only direct dependencies. If the *`depth`* parameter is omitted, the search is unbounded.
**Example**
The following example, returns the targets in the [transitive closure](https://en.wikipedia.org/wiki/Transitive_closure) of `//foo:bar` that depend directly on `//example:baz`.

```
buck query "rdeps('//foo:bar', '//example:baz', 1)"
```

**Example**
The *universe:expr* parameter includes the *entire* transitive closure of the target pattern specified. So some of these targets might be outside the directory structure indicated by that target pattern. For example, the result set of

```
buck query "rdeps('//foo/bar/...', '//fuga:baz', 1)"
```

might contain targets outside the directory structure beneath

```
foo/bar/
```

To say it another way, if a target in `//foo/bar/...` depends on, say, `//hoge,` which in turn depends on `//fuga:baz,` *then `//hoge` would show up in the result set*.
If you wanted to constrain the result set to only those targets beneath `foo/bar`, you could use the [`intersect`](https://buck.build/command/query.html#set-operations) operator:

```
buck query "rdeps('//foo/bar/...', '//fuga:baz', 1) ^ '//foo/bar/...'"
```

The caret (`^`) is a succinct synonym for `intersect`.

#### List the tests of the specified targets: testsof

**Syntax**

```
testsof(*expr*)
```

The `testsof(`*`expr`*`)` operator returns the tests associated with the targets specified by the target expression, *`expr`*. For example,

```
buck query "testsof(set('//foo:bar' '//baz:app+lib')"
```

returns the tests associated with `//foo:bar` and `//baz:app+lib`.
To obtain all the tests associated with the target and its dependencies, you can combine the `testsof()` operator with the `deps()` operator. For example,

```
buck query "testsof(deps('//foo:bar'))"
```

first finds the transitive closure of `//foo:bar`, and then lists all the tests associated with the targets in this transitive closure.

## Executing multiple queries at once

Suppose you want to know the tests associated with a set of targets. This can be done by combining the `testsof`, `deps` and `set` operators. For example,

```
buck query "testsof(deps(set('target1' 'target2' 'target3')))"
```

Suppose you now want to know the tests for *each* of these targets; the above command returns the union of the tests. Instead of executing one query for the entire set of targets, `buck query` provides a way to repeat a query with different targets using a single command. To do this, first define the query expression format and then list the input targets, separated by spaces. For example,

```
buck query "testsof(deps( %s ))" target1 target2 target3
```

The `%s` in the query expression is replaced by each of the listed targets, and for each target, the resulting query expression is evaluated. If you add the `--output-format json` parameter, the result of the command is grouped by input target; otherwise, as in the previous example using `set()`, the command merges the results and returns the union of the queries.
This syntax is also useful for subcommands that take arguments that are not targets, such as `owner()`. Recall that the `set()` operator works only with targets, but the `owner()` operator takes a filename as its argument.

```
buck query "owner( %s )" main.cpp myclass.cpp myclass.h
```

## Referencing Args Files

When running queries, arguments can be stored in external files, one argument per line, and referenced with the `@` symbol. This is convenient when the number of arguments is long or when you want to persist the query input in source control.

```
buck query "testsof(deps(%s))" @/path/to/args-file
```

If you want to include all the targets in the `@`-file in a single query execution, you can use the following alternative syntax. Note the addition of the capital “`S`" in "`%Ss`".

```
buck query "testsof(deps(%Ss))" @/path/to/args-file
```

In the example above, the lines of the file are converted to a set and substituted for the `%Ss`. In addition, each line's contents are singly quoted. In the example above, if the args file contains the following:

```
//foo:bar
//foo:baz
```

Then the query expression is equivalent to:

```
buck query "testsof(deps(set('//foo:bar' '//foo:baz')))"
```

If you use multiple `%Ss` operators in a single query, you can specify which lines in the `@`-file should be used for each instance of `%Ss` in the query expression: use `--` to separate elements that go in different sets. For example:

```
buck query "testsof(deps(%Ss)) union deps(%Ss)" @path/to/args-file
//foo:foo
--
//bar:bar
```

is equivalent to running the following:

```
buck query "testsof(deps(set('//foo:foo'))) union deps(set('//bar:bar'))"
```

## Parameters

* `--output-format dot`
    Outputs the digraph representing the query results in [dot format](https://en.wikipedia.org/wiki/DOT_(graph_description_language)#Directed_graphs). The nodes will be colored according to their type. See [graphviz.org](http://www.graphviz.org/doc/info/colors.html) for color definitions.

```
android_aar          : springgreen2
android_library      : springgreen3
android_resource     : springgreen1
android_prebuilt_aar : olivedrab3
java_library         : indianred1
prebuilt_jar         : mediumpurple1
```

Example usage:

```
$ buck query "allpaths('//foo:bar', '//path/to:other')" --output-format dot --output-file graph.dot
$ dot -Tpng graph.dot -o graph.png
```

Then, open `graph.png` to visualize the graph.

* `--output-format dot_bfs`
    Outputs the digraph representing the query results in [dot format](https://en.wikipedia.org/wiki/DOT_(graph_description_language)#Directed_graphs) in bfs order. The nodes will be colored according to their type.Example usage:

```
$ buck query "allpaths('//foo:bar', '//path/to:other')" --output-format dot_bfs --output-file graph.dot
$ dot -Tpng graph.dot -o graph.png
          
```

Then, open `graph.png` to visualize the graph.

* `--output-format json`
    Outputs the results as JSON.
* `--output-format thrift`
    Outputs the results as thrift binary.
* `--output-file`
    Outputs the results into file path specified.Example usage:

```
$ buck query "allpaths('//foo:bar', '//path/to:other')" --output-format dot --output-file graph.dot
$ dot -Tpng graph.dot -o graph.png
          
```

* `--output-attributes <attributes>`$ buck query '//example/...' —output-attributes buck.type name srcs{"//example/foo:bar" : {"buck.type" : "cxx_library","name" : "foobar","srcs" : [ "example/foo/bar.cc", "example/foo/lib/lib.cc" ]}"//example/foo:main" : {"buck.type" : "cxx_binary","name" : "main"}}`--output-attribute <attribute>`$ buck query '//example/...' —output-attribute buck.type —output-attribute name —output-attribute srcs{"//example/foo:bar" : {"buck.type" : "cxx_library","name" : "foobar","srcs" : [ "example/foo/bar.cc", "example/foo/lib/lib.cc" ]}"//example/foo:main" : {"buck.type" : "cxx_binary","name" : "main"}}

## Examples

```
## For the following examples, assume this BUCK file exists in
# the `examples` directory.
#


cxx_library(
  name = 'one',
  srcs = [ '1.cpp' ],
  deps = [':two',':three',],)

cxx_library(
  name = 'two',
  srcs = [ '2.cpp' ],
  deps = [':four',],
  tests = [ ':two-tests' ])

cxx_library(
  name = 'three',
  srcs = [ '3.cpp' ],
  deps = [':four',':five',],
  tests = [ ':three-tests' ],)

cxx_library(
  name = 'four',
  srcs = [ '4.cpp' ],
  deps = [':five',])

cxx_library(
  name = 'five',
  srcs = [ '5.cpp' ],)

cxx_test(
  name = 'two-tests',
  srcs = [ '2-test.cpp' ],
  deps = [ ':two' ],)

cxx_test(
  name = 'three-tests',
  srcs = [ '3-test.cpp' ],
  deps = [ ':three' ],)buck query "//..."//examples:five
//examples:four
//examples:one
//examples:three
//examples:three-tests
//examples:two
//examples:two-testsapp = //apps/myapp:app
lib = //libraries/mylib:libbuck query "%s" app lib —output-format json{"app": ["//apps/myapp:app"],"lib": ["//libraries/mylib:lib"]}$ buck query "deps(//examples:one, 1)"
//examples:one
//examples:three
//examples:two$ buck query —output-format json "deps(//examples:one)"
[
  "//examples:five",
  "//examples:four",
  "//examples:one",
  "//examples:three",
  "//examples:two"
]$ buck query —output-format json "testsof(deps('%s'))" //examples:one //examples:three
{
  "//examples:one": ["//examples:two-tests"],
  "//examples:three": ["//examples:three-tests"]
}$ buck query "buildfile(owner('examples/1.cpp'))"
example/BUCK
```



# buck run

This command builds and runs an executable resulting from building a target.

```
buck run //app:app-dist
```

For Android/iOS rules which can't be run directly, [`buck install -r`](https://buck.build/command/install.html) should be used to install the result of the build on the appropriate emulator/simulator.

## Common Parameters

All the parameters for [`buck build`](https://buck.build/command/build.html) also apply to `buck run`.

## Passing Arguments

Passing the `--` flag will cause all following arguments to be piped through to the binary called by `buck run`. For example, calling

```
buck run //:bin --no-cache -- --foo=bar arg1 arg2
```

Will build `//:bin` without the cache and call the result with the arguments `["--foo=bar", "arg1", "arg2"]`


# buck root

A command that prints out the absolute path to the root directory of the current buck project. This is useful from within scripts to help with navigating the project tree.

```
buck root
/home/buck/example
```

## Parameters

This command takes no parameters.


# buck server

Find information about the http server run by Buck.

```
buck server status --http-port
```

## Commands

* `status --http-port` If the buck daemon is running an http server, returns the port that the server is running on. Else, returns `-1`.

## Parameters

* `--json` Outputs the results as JSON.



# buck targets

List information about the build targets in the current project.

```
buck targets <[build target](https://buck.build/concept/build_target.html)> | <[build target pattern](https://buck.build/concept/build_target_pattern.html)> ...
```

The `buck targets` command must take at least one parameter: a [build target](https://buck.build/concept/build_target.html) or [build target pattern](https://buck.build/concept/build_target_pattern.html) or a combination of these.
The following prints all build targets in the project (sorted alphabetically) to standard out:

```
buck targets //...
```

You can pass a list of targets to `buck targets`, and Buck prints information only for those targets. For example, the following command line prints the target `main` from the `BUCK` file in the root of the Buck project and all the targets from the `BUCK` file in the subdirectory `myclass`.

```
buck targets //:main //myclass:
```

The `buck targets` command is commonly used with the `--show-output` option to obtain the output paths for the build artifacts for Buck targets.

```
buck targets --show-output //java/com/myproject:binary
> //java/com/myproject:binary buck-out/gen/java/com/myproject/binary.apk
```

## Parameters

* `--type`
    The types of target by which to filter. For example:

```
buck targets //java/com/myproject/... --type java_test java_binary
```

Note that the `--type` parameter comes *after* the specified targets.
This parameter can be handy in programmatic tasks, such as running all of the Java tests under `//java/com/myproject`:

```
buck targets //java/com/myproject/... --type java_test | xargs buck test
```

* `--referenced-file`
    Filters targets by the list of rules that include the specified file in their transitive closure.
    For example, to run all tests that could be affected by a particular file, you could use:

```
buck targets //java/com/myproject/... --type java_test \
  --referenced-file java/com/example/Foo.java |
  xargs buck test
```

* `--json`
    Print a JSON representation of each target.
    In addition, the JSON includes a list of `direct_dependencies` for each target, which may include additional dependencies for targets whose descriptions implement `ImplicitDepsInferringDescription`. The fully qualified names of targets are given.
    For example, when resolving a genrule, the direct dependencies includes both the build targets in `deps` as well as any build targets in a script associated with the genrule.
* `--output-attributes`
    Specify the attributes used in the JSON representation.
    If you omit this option, Buck shows all attributes.
* `--print0`
    Delimit targets using the ASCII NUL character if `--json` is not specified. This facilitates use with `xargs`:

```
buck targets --print0 | xargs -0 buck build
```

* `--resolve-alias`
    Print the fully-qualified build target for the specified alias[es]. This command also accepts build targets. For more information, see [`[alias]`](https://buck.build/files-and-dirs/buckconfig.html#alias) in the `.buckconfig` documentation.
* `--show-output`
    Print the relative paths to the output for each target after the target name. Note that some rules—such as [`genrule`](https://buck.build/rule/genrule.html)—generate sources instead of build artifacts.
* `--show-rulekey`
    Print the [rule keys](https://buck.build/concept/rule_keys.html), for the specified targets.
    For example, if you have a rule named `main` defined in the build file in your current directory, the following command prints its rule key.

```
buck targets --show-rulekey ':main'
```

Note that the rule key for a target is different from the *target hash* for that target. (See `show-target-hash` below.) Further, the inputs for the computation of the rule key are *not* a superset of the inputs for the target hash. For example, the value of the [visibility](https://buck.build/concept/Visibility.html) argument *is not* an input for the rule key but *is* an input for the target hash.

* `--show-transitive-rulekeys`
    When specified in conjunction with `--show-rulekey`, prints the [rule keys](https://buck.build/concept/rule_keys.html) for the specified targets *and their transitive closure*.
    For example, if you have a rule named `main` defined in the build file in your current directory, the following command prints the rule key for that rule and the rule keys for all of the rules that are in its transitive closure.

```
buck targets --show-rulekey --show-transitive-rulekeys ':main'
```

* `--show-target-hash`
    Print each rule's target hash after the rule's name. Target hashes can be used to detect which targets are affected when source files in BUCK files change: if a target is affected, its target hash will be different after the change from what it was before.
    A target hash is created by finding all the transitive dependencies of the given target and hashing all their attributes and the files they reference. For more details about how the referenced files are hashed see the `--target-hash-file-mode` flag. *The format of the data that is hashed is undocumented and can change between Buck versions.*
* `--target-hash-file-mode`
    Modify how target hashes are computed.
    Can be set to either `PATHS_AND_CONTENTS` or `PATHS_ONLY`.
    If set to `PATHS_AND_CONTENTS` (the default), the contents of all files referenced from the targets will be used to compute the target hash.
    If set to `PATHS_ONLY`, only files' paths contribute to the hash. `PATHS_ONLY` will generally be faster because it does not need to read all of the referenced files, but it will not detect file changes automatically. See `--target-hash-modified-paths` for another way to handle changes to referenced files without losing the performance benefits.
* `--target-hash-modified-paths`
    Modify how target hashes are computed.
    If a target or its dependencies reference a file from the specified set of paths, the target's hash will be different than if this option had been omitted. Otherwise, the target's hash will be the same as if this option was omitted.
    This option can be used to detect changes in referenced files if the list of modified files is available from an external source, such as a source control system.
    This option is effective only when `--target-hash-file-mode` is set to `PATHS_ONLY`. Otherwise, the actual contents of the files are used to detect modifications and this option is ignored.



# buck test

Builds and runs the tests for one or more specified targets:

```
buck test //javatests/com/example:tests
```

You can either directly specify test targets, or any other target which contains a `tests = ['...']` field to specify its tests.

## Parameters

* `--all` Run all tests available in the tree. If no targets are specified, this is the default.
* `--code-coverage` Collects code coverage information while running tests. Currently, this only works with Java using [JaCoCo](http://www.eclemma.org/jacoco/). After running:

```
buck test --code-coverage
```

The code coverage information can be found in:

```
buck-out/gen/jacoco/code-coverage/
```

* `--debug` If specified, tests will start suspended and will not run until a debugger is attached. Tests compatible with JDWP will be listening on the default port (5005), lldb tests print out a process ID to attach to.
* `--include` Test labels to run with this test. Labels are a way to group together tests of a particular type and run them together. For example, a developer could mark all tests that run in less than 100 milliseconds with the `fast` label, and then use:

```
buck test --all --include fast
```

to run only fast tests. See [`java_test()`](https://buck.build/rule/java_test.html) for more details.
Use multiple arguments to match any label, and `+` to match a set of labels. For example to match all the fast tests that are either stable or trustworthy, and aren't unstable:

```
… --include fast+stable fast+trustworthy --exclude fast+unstable
```

* `--exclude` The inverse of `include`. Labels specified with the exclude option won't be run. For example, if we wanted to run all tests except slow ones, we would run:

```
buck test --all --exclude slow
```

* `--test-selectors` `(-filter)` Select tests to run by name, using a `class#method` syntax. All other tests will not be run and test result caching is disabled:

```
buck test --all --test-selectors 'com.example.MyTest#testX'
```

Matching is done using `java.util.regex` regular expressions, and the class part (or method) part can be omitted to match all classes (or methods). Selectors are anchored to the end of each class and/or method name (i.e. a `$` at the end of your regular expressions is implied.)

```
buck test --all --filter 'Foo.*'  # ...every class starting Foo
buck test --all --filter '#testX' # ...run testX in every class
```

You can exclude tests with `!`, and if all your test selectors are exclusive, then the default is to run everything except those tests:

```
buck test --all --test-selectors '!MyTest'  # ...all except MyTest
```

Test selectors can also be read from a file by formatting the command line argument as `:/path/to/file`. The file should contain one test selector per line.
The first matching selector decides whether to include or exclude a test. The full logic is described in the `--help`.

* `--num-threads` The number of threads that buck should use when executing the build. This defaults to 1.25 times the number of processors in the system (on systems with hyperthreading, this means that each core is counted twice). The number of active threads may not always be equal to this argument.
* `--ignore-when-dependencies-fail` If a library is broken its tests are probably failing. If another library depends on that library and its tests are also failing, it is probably because the dependency has a bug.
    For example, if the library `HouseBuilder` depends on `Bricks` and the `Bricks` library is broken, it will probably cause its own tests as well as `HouseBuilder`'s to fail.
    Accordingly, if the libraries are tested respectively by `HouseBuilderTest` and `BricksTest`, and both tests fail then only the error for `BricksTest` is printed; the error for `HouseBuilderTest` is ignored.
    You'll still be notified that `HouseBuilderTest` is failing, and running the tests again without this option will show the cached test result (and error) in full.
* `--test-runner-env` Add or override an environment variable passed to the test runner. Can be specified multiple times for different environment variables. Later occurrences override earlier occurrences. Currently this only support Apple(ios/osx) tests.

```
buck test --test-runner-env FOO=BAR --test-runner-env BAZ=QUUX //some:target
```

* `--verbose` `(-v)` How verbose logging to the console should be, with 1 as the minimum and 10 as the most verbose.
* `--xml` If specified, Buck will write the test results as XML to the location specified. For example:

```
buck test --all --xml testOutput.xml


```



# buck uninstall

Uninstalls the APK for an [`android_binary`](https://buck.build/rule/android_binary.html) target.
Takes an [`android_binary`](https://buck.build/rule/android_binary.html) target, determines the id of the application that it is meant to build, and uninstalls it by running `adb uninstall <application_id>`:

```
buck uninstall //java/com/myawesomeapp
```

## Parameters

None.




# Exit Codes

These exit codes are returned from a Buck command to the shell when the command exits.
These exit codes are Buck's *binary protocol* for interacting with other software such as shell scripts.
Note that in the case of some fatal errors—such as `FATAL_OOM`, `FATAL_IO`, or `FATAL_DISK_FULL`—Buck itself might not be able to *reliably* detect the source of the failure. If this occurs, Buck falls back to reporting `FATAL_GENERIC`.

|0	|`SUCCESS`	|The command returned successfully. No errors. No warnings.	|
|---	|---	|---	|
|1	|`BUILD_ERROR`	|Build resulted in a non-specific user error.	|
|2	|`BUSY`	|Buck daemon is busy processing another command. For more information, see [**Buck Daemon (buckd)**](https://buck.build/command/buckd.html).	|
|3	|`COMMANDLINE_ERROR`	|Incorrect user-supplied command-line options. For more information, see [**Common Parameters**](https://buck.build/command/common_parameters.html) or the topic page for the specific command that you executed.	|
|4	|`NOTHING_TO_DO`	|Nothing to build or evaluate for the specified command.	|
|5	|`PARSE_ERROR`	|Error in parsing the build file or in constructing the target or action graph.	|
|6	|`RUN_ERROR`	|Failure while running a binary or installing a binary on a device. For more information, see [`buck run`](https://buck.build/command/run.html).	|
|10	|`FATAL_GENERIC`	|Generic non-recoverable internal error.	|
|11	|`FATAL_BOOTSTRAP`	|Non-recoverable error in Buck bootstrapper.	|
|12	|`FATAL_OOM`	|Non-recoverable out-of-memory (OOM) error.	|
|13	|`FATAL_IO`	|Generic non-recoverable I/0 error.	|
|14	|`FATAL_DISK_FULL`	|No space on storage device.	|
|32	|`TEST_ERROR`	|Test run had user-specific test errors. For more information, see [`buck test`](https://buck.build/command/test.html).	|
|64	|`TEST_NOTHING`	|There were no tests to run. For more information, see [`buck test`](https://buck.build/command/test.html).	|
|130	|`SIGNAL_INTERRUPT`	|Command was interrupted (Ctrl + C)	|

