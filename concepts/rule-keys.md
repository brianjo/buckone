# Rule Keys

Buck works on top of build dependency graphs to decide what commands to run and what order to execute them in. The picture in this page depicts the last graph used by Buck during a build. This graph is called the "Action Graph" and each node is referred to as a BuildRule.
RuleKeys are used to encapsulate all of the factors which may contribute to the output of a BuildRule into a single value - a hashcode. If any RuleKey of a BuildRule has not changed, the output of that BuildRule cannot have changed.
The goal of the RuleKeys is to help buck minimise the amount of BuildRules that need to be built locally on each buck run. The artifacts stored in the buck caches (both HTTP and dircache) are unsurprisingly keyed off these RuleKeys as well.

## Common Rule Key Computations

There are different RuleKey types, each allowing for different optimizations.
All RuleKeys are hashes of some information. The following base set of information is included in all RuleKeys:

* **Target type** (e.g. java_library) - this defines the actual computation that will produces the output of the BuildRule, such as what processes are executed.
* **Buck version** - the internal implementation details of what a BuildRule means may change across Buck versions.
* **Target name and package** (e.g. //example:foo) - this is used in things like generated output file names.
* **The buck-out directory relative to the root of the Buck project** - this is used in input and output paths.

Certain other information is also recorded on a per-target-type basis, e.g. environment variables which may affect the output, toolchain versions, etc. Each different RuleKey type will then add extra information to the hash. The RuleKey types Buck currently supports are:

## Rule Key Types

There are 4 types of Rule Keys in buck:

1. [Input Based Rule Keys](https://buck.build/concept/rule_keys.html#input_based_rule_keys)
2. [Default Rule Keys](https://buck.build/concept/rule_keys.html#default_rule_keys)
3. [ABI Rule Keys](https://buck.build/concept/rule_keys.html#abi_rule_keys)
4. [Dependency File RuleKey (or "Manifest RuleKey")](https://buck.build/concept/rule_keys.html#manifest_based_rule_keys)

### Input Based Rule Keys

These are the simplest RuleKeys to reason about. They are always applicable in all situations. Unfortunately, they require the dependencies of the BuildRule to have already been built. This RuleKey adds the following information to the hash:

* The relative path to, and contents of, each direct input file to the BuildRule (typically, the value of the srcs attribute).
* The relative path to, and contents of, each output file of each direct dependency of the BuildRule (typically, the output files of the values of the deps attribute).

**Example: **Let's say buck is building BuildRule #2 from the example Action Graph. The input based rule key for this node will be generated out of the combined hashes of all srcs defined in #2 with the hashes of the outputs of BuildRules #4 and #5.

### Default Rule Keys

These allow a RuleKey to be computed which is always applicable, without needing to actually build the BuildRule's dependencies. This optimises for builds which have high cache hit rates, as it allows Buck to optimistically check caches from the top of our dependency graph, iterating down until it gets a hit, and then only execute the BuildRules which missed the cache, rather than needing to fetch cache hits (or execute BuildRules) all the way from the bottom of the graph. This RuleKey adds the following information to the hash:

* The relative path to, and contents of, each direct input file to the BuildRule (typically, the value of the srcs attribute).
* The default RuleKey value for each direct dependency of the BuildRule (typically, the default RuleKeys of the BuildRules which are the values of the deps attribute. Some rules may also have dependencies other than those listed in deps, such as on compiler toolchains).

**Example: **Once more taking as an example BuildRule #2, in order to compute its rule key buck will compute the hash of its direct srcs dependencies and combine that with the default rule keys of #4 and #5. Notice recursive nature of this process, as the default rule key for #4 can only be calculated off the computation of the default rule key for #7.

### ABI Rule Keys

In some cases, only part of the output of a BuildRule is used by the things which depend on it. For example, a java_library rule does not need to be rebuilt in the case that one of its dependencies changes if the public interface of the dependency didn't change. This knowledge can be used to avoid spurious rebuilds, where it is known that the output will be the same.
Note, though, that this RuleKey is not necessarily transitively applicable - in the Java example, the java_binary which eventually depends on the java_library *will* still need to be re-packaged to pick up the changes which occurred in its transitive dependencies. It is important to only apply ABI RuleKeys where they are applicable. This RuleKey adds the following information to the hash:

* The relative path to, and contents of, each direct input file to the BuildRule (typically, the value of the srcs attribute).
* A representation of the ABI (e.g. Java public interface) of each direct dependency of the BuildRule (typically, of the values of the deps attribute).

Not all BuildRule types support ABI RuleKeys.
**Example:** Let's use again BuildRule #2 as an example. In order to compute the ABI Rule Key, buck will compute the hashcode of #2's direct source dependencies (srcs) and will combine that with the ABI representation of the output of BuildRules #4 and #5.

### Dependency File RuleKey (or "Manifest RuleKey")

In some cases, the listed input files to a BuildRule are an over-approximation of the things which actually affect compilation.
For example, a cxx_library rule may list some header files which are not actually used by all of the source files it compiles. In this case, spurious recompiles of those source files can be avoided if they are not affected by the changed header file. The knowledge about which headers are depended on is only known after compilation has occurred, so there is a bootstrapping build (or cache hit) which is required before this RuleKey can be used. This over-approximation could be avoided, but for the sake of usability we allow this, and introduce optimizations to avoid re-executing BuildRules in this case.
To compute this RuleKey, two additional lists are required;

1. A list of files which are conditional inputs (e.g. for a cxx_library, the list of header files)
2. A list of the conditional input files which were actually used in the most recent BuildRule execution

Note that this RuleKey is only applicable where the conditionality of the usage of any particular input could not have changed since these lists were constructed (e.g. for a cxx_library, if no #include statements were added, removed, modified, or re-ordered, and no header files were added or removed to the target). This RuleKey adds the following information to the hash:

* The relative path to, and contents of, each direct input file to the BuildRule which is not in the conditional input files list (i.e. which is an unconditional input).
* The relative path to, and contents of, each direct input file to the BuildRule which was in the list of conditional input files which were actually used.

Note that Dependency File RuleKeys are not currently implemented in such a way as to take any other dependencies into account; all dependencies are assumed to be expressed in the above information.
Not all BuildRule types support Dependency File RuleKeys.
