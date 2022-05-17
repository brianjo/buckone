# What Makes Buck so Fast?

Buck exploits a number of strategies to reduce build times.

## Buck builds dependencies in parallel

Buck is designed so that any input files required by a [build target](https://buck.build/concept/build_target.html) must be specified in the [build rule](https://buck.build/concept/build_rule.html) for that target. Therefore, we can know that the directed acyclic graph [(DAG)](http://en.wikipedia.org/wiki/Directed_acyclic_graph) that Buck constructs from the build rule is an accurate reflection of the build's dependencies, and that once a rule's dependencies are satisfied, the target for that rule can be built.
Having a DAG makes it straightforward for rules to be built in parallel, which can dramatically reduce build times. Buck starts with the leaf nodes of the graph, that is, targets that have no dependencies. Buck adds these to a queue of targets to build. When a thread is available, Buck removes a target from the front of the queue and builds it. Assuming the target builds successfully, Buck notifies all of the rules that depend on that target. When all of a rule's dependencies have been satisfied, Buck adds that rule's target to the build queue. Computation proceeds in this manner until all of the nodes in the graph have been built. This execution model means that breaking modules into finer dependencies creates opportunities for increased parallelism, which improves throughput.

## Graph enhancement increases rule granularity

Frequently, the granularity at which users declare build rules is different from the granularity at which we want the build system to model them. For simplicity, users want coarse-grained rules, such as [`android_binary`](https://buck.build/rule/android_binary.html). However, the build system wants fine-grained rules, such as `AaptPackage` and `DexMerge`, that allow for more parallelism and more granular caching. See the section on *caching* below.
Internally, Buck uses a mechanism called *graph enhancement* which transforms the *target graph*, specified by the build rules, into an *action graph*, which is the DAG that Buck actually uses for building. Graph enhancement can add new synthetic rules to break a monolithic task, such as `android_binary` into independent subtasks, each of which might have only a subset of the original task's dependencies. So, for example, dex merging would not depend on a full run of `AaptPackage`. Graph enhancement can also move dependency edges, so that compiling Android libraries does not depend on dexing their dependencies.
**Note:** Adding or removing dependencies from your build causes Buck to rebuild the action graph, which can significantly increase the time required for your next build. However, changing the *contents* of a dependency, such as a source file, does not cause Buck to rebuild the action graph.

## Buck caches build artifacts

A build rule—together with other aspects of the build environment—specify all of the inputs that can affect the rule's output. Therefore, we can combine that information into a hash that represents the totality of those inputs. This hash—called a [*RuleKey*](https://buck.build/concept/rule_keys.html)—is used as an index or *cache key* into a cache where the associated value is the output produced by the rule. (See [`.buckconfig`](https://buck.build/files-and-dirs/buckconfig.html) for information on how to set up Buck's caches.) Some of the factors that affect the RuleKey for a build rule are:

* The contents of any file arguments to the build rule. For example, the files specified in the `srcs` or `headers` arguments. If you use the [`glob()`](https://buck.build/function/glob.html) function in these arguments, to pattern-match files, be aware that `glob()` could potentially pull in extraneous files generated by, for example, text editors or the operating system. These files could then cause unexpected variations in RuleKeys between different development computers.
* The RuleKey for each of the rule's *dependencies*. By dependencies, we mean build targets specified in the rule's arguments. Note that you can specify build targets in arguments other than just the `deps` argument. For example, you can specify build targets in the `srcs` or `headers` arguments. An expression in a `deps_query` argument can also return a set of build targets.
* Some rules are defined using [macros](https://buck.build/extending/macros.html). Therefore, changes to a macro can cascade through to the definition of a rule and therefore change the rule's RuleKey.
* The build environment, which includes the components of the toolchain that are used to build the rule and the configurations of those tools, such as compiler flags or linker flags. The build configuration is a function of—for example—the rule itself, the [`.buckconfig`](https://buck.build/files-and-dirs/buckconfig.html) and `.buckconfig.local` files, Buck's command-line parameters, and the defaults of the local environment.
* The version of Buck used to build the rule. This means that upgrading Buck to a new version invalidates all of the RuleKeys generated by the old version.

When Buck determines whether to build the target for a rule, it first computes the RuleKey for the rule. If that key results in a hit in any of the caches specified in `.buckconfig`, Buck fetches the rule's output from the cache instead of building the rule locally. For outputs that are expensive to build, this results in substantial savings. This caching can also make it fast to rebuild when switching between branches in a [DVCS](http://en.wikipedia.org/wiki/Distributed_version_control_system) such as Git or Mercurial—assuming that relatively few files differ between branches.
If you are using a [continuous integration (CI)](http://en.wikipedia.org/wiki/Continuous_integration) system, such as [Jenkins](https://en.wikipedia.org/wiki/Jenkins_(software)), you should configure your CI builds to populate a cache that can be read by local builds. That way, when a developer syncs to a revision that has already been built by your CI system, a local build with [`buck build`](https://buck.build/command/build.html) can pull build artifacts from the cache. In order for this strategy to work, the RuleKeys computed by Buck on the CI system must match the corresponding keys computed by Buck on the developer's local computer.

## The importance of deterministic builds

In order to take full advantage of caching, all the factors that affect the output of the build should be kept safe from unintended changes. The build should be *deterministic* in the sense that it should reliably produce identical output across different build servers or different developers' computers. For this reason, we recommend that you keep all inputs to the build under source control. These would include, for example, Buck's configuration file, `.buckconfig`, and the toolchains used to build the outputs, such as compilers and linkers. This way, you can ensure that all developers on the project have the same build environment.

### Command-line configuration changes

Consistent with the preceding discussion, Buck reparses the build files in a project if it detects certain changes in the build's configuration. These could be changes in the `.buckconfig` file itself or be the result of *specifying configuration parameters with the *[--config](https://buck.build/command/common_parameters.html) *command-line option.*

## If a Java library's API doesn't change, code that uses the library doesn't need to be rebuilt

Developers often modify a Java library in ways that do not affect the library's externally-visible API. For example, adding or removing private methods, or modifying the implementation of existing methods—regardless of their visibility—does not change the API exposed by the Java library.
When Buck builds a [`java_library`](https://buck.build/rule/java_library.html) rule, it also computes that library's API. Normally, modifying a private method in a [`java_library`](https://buck.build/rule/java_library.html) would cause the library and all the rules that depend on it to be rebuilt because the change in RuleKeys would propagate up the DAG. However, Buck has special logic for [`java_library`](https://buck.build/rule/java_library.html) where, if the `.java` input files have not changed since the previous build, and the API for each of its Java dependencies has not changed since the previous build, then the [`java_library`](https://buck.build/rule/java_library.html) will not be recompiled. This is valid because we know that neither the input `.java` files nor the API against which they would be compiled has changed, so the result would be the same if the rule were rebuilt. This localizes how much Java code needs to be recompiled in response to a change, again reducing build times.
For more information about this Java Library optimization, see [Java Application Binary Interfaces (ABIs)](https://buck.build/concept/java_abis.html).

## RuleKeys and input-based keys

As a generalization of the Java library optimization—described in the previous section—other rule types have functionality to determine whether or not to rebuild themselves based on information about the state of their dependencies—irrespective of whether those dependencies have changed.
For example, if we change a file that is an input to an [`android_resource`](https://buck.build/rule/android_resource.html) rule, we don't need to recompile targets that depend on the resource if the set of exposed symbols hasn't changed—such as the case where we just change a padding value. Similarly, if we recompile an [`android_library`](https://buck.build/rule/android_library.html) due to a dependency change, but the resulting classes are identical, we don't need to re-run the DEX compiler (`dx`).
The determination about whether to rebuild is based on the value of an additional key: an *input-based* key, which is distinct from the standard RuleKey that Buck generates for the rule. For a standard RuleKey, Buck folds in the RuleKey for each dependency, but for an input-based key, Buck folds in a hash of the actual output file from the dependency. For example, suppose `//:foo` specifies `//:bar` as a dependency. The *RuleKey* for `//:foo` folds in the RuleKey for `//:bar`, but the input-based key for `//:foo` folds in a hash of the output from `//:bar` that `//:foo` takes as input.

## Buck uses only first-order dependencies for Java

When compiling Java, Buck uses first-order dependencies only, that is, dependencies that you specify explicitly in the `deps` argument of your build rule. This means that the compilation step in your build sees only explicitly-declared dependencies, not other libraries that those dependencies themselves depend on.
Using only first-order dependencies dramatically shrinks the set of APIs that your Java code is exposed to, which dramatically reduces the scope of changes that will trigger a rebuild.
**NOTE:** If your rule does, in fact, depend on a dependency of one of your explicitly-specified dependencies—such as a *second-order* dependency—you can make that dependency available to your rule by specifying it in an `exported_deps` argument in the rule of the explicitly-specified dependency.

## Buck uses dependency files to trim over-specified inputs

Buck's low-level build rules specify all inputs—such as source files or the outputs from other build rules—that might contribute to the output when the build rule is executed. Normally, changes to any of these inputs result in a new RuleKey and therefore trigger a rebuild. However, in practice, it's not uncommon for these build rules to *over-specify* their inputs. A good example is Buck's C/C++ compilation rules. C/C++ compilation rules specify as inputs all headers found from the transitive closure of C/C++ library dependencies, even though in many cases only a small subset of these headers are actually used. For example, a C/C++ source file might use only one of many headers exported by a C/C++ library dependency. However, there's not enough information available before running the build to know if any given input is used, and so all inputs must be considered, which can lead to unnecessary rebuilding.
In some cases, after the build completes, Buck can figure out the exact subset of the listed inputs that were actually used. In C/C++, compilers such as `gcc` provide a `-M` option which produces a dependency file. This file identifies the exact headers that were used during compilation. For supported rules, Buck uses this dependency file before the build, to try to avoid an unnecessary rebuilding:

* If the dependency file is available before the build, Buck reads the file and uses it to filter out unused inputs when constructing the RuleKey.
* If no dependency file is available before the build, Buck runs the build as normal and produces a dependency file. The dependency file is then available for subsequent builds.

Note that dependency files are used only if the standard RuleKey—which considers all inputs—doesn't match. In cases where the RuleKey matches, the output from the rule can be fetched from the cache.