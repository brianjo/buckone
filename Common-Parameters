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
