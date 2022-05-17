# Java ABIs

This topic pertains to building Java code with Buck.
When compiling a Java rule, Buck creates an *Application Binary Interface (ABI) JAR*, which contains only resources and class interfaces, that is, the public interface for your module. Buck creates this ABI JAR in addition to the *library JAR*, which contains all of the compiled classes and resources for the rule. Since ABI JARs do not contain method bodies or private members, they are smaller and change less frequently than library JARs, which enables Buck to use ABI JARs in two ways:

* Use them in [ABI rule keys](https://buck.build/concept/rule_keys.html#abi_rule_keys) to more accurately determine which rules need to be rebuilt during an incremental build. In some cases, only part of the output of a BuildRule is used by the things which depend on it. For example, a java_library rule does not necessarily need to be rebuilt if one of its dependencies changes, *provided that the public interface of that dependency did not change.* This knowledge can be used to avoid extraneous rebuilds, where it is known that the output will be the same.
* Use them on the compiler's classpath instead of library JARs to get a small but significant performance boost because the smaller size of ABI JARs enables the compiler to load them faster than library JARs.

## Three Kinds of ABI JARs

Depending on the [`[java].abi_generation_mode`](https://buck.build/files-and-dirs/buckconfig.html#java.abi_generation_mode) config setting, Buck can create an ABI JAR in three ways:

* From classes, by building the library JAR first and then stripping out the unnecessary bits.
* From source, by hooking in to the compiler while it is compiling the library JAR and emitting the ABI JAR partway through the compilation process. This helps to reduce bottlenecks due to slow rules or low parallelism in a build graph.
* From source *only*, by examining only the text of the source code, and using heuristics to infer things that can normally be determined only by looking at dependencies. This dramatically increases parallelism in the rule graph and reduces the number of cache fetches required during incremental builds. The next section goes into the requirements for these *source-only ABI JARs* in greater detail.

## Requirements for Source-only ABI JARs

Buck generates source-only ABI JARs using only the text of the source code for a rule, without first compiling (most of) the rule's dependencies. Some details of an ABI JAR cannot be known for certain from just the source, so Buck uses heuristics to infer those details. Even when using source-only ABIs, Buck still uses a rule's dependencies to compile the library JAR. At that time, Buck verifies whether the heuristics used for the ABI JAR were correct, and if they were not, Buck fails the build with an error.

Such build failures can generally be fixed with [`buck fix`](https://buck.build/command/fix.html), although sometimes these fixes are sub-optimal. In most cases, [`buck fix`](https://buck.build/command/fix.html) adds—or suggests that you add—either `source_only_abi_deps=[<dependencies>]` or `required_for_source_only_abi=True`. However, we recommend that, if possible, you avoid using `source_only_abi_deps` and `required_for_source_only_abi`.

Both `source_only_abi_deps` and `required_for_source_only_abi` have the potential to negatively impact build times. The list of dependencies specified by `source_only_abi_deps` must be built before Buck can build a source-only ABI for the current module. In the case of `required_for_source_only_abi`, the current module must be built first before source ABIs that depend on it can be built. Whereas `source_only_abi_deps` affects only the current module, `required_for_source_only_abi` affects every module that depends on the current module.

The following points describe the requirements for source-only ABI generation, specifically how [`buck fix`](https://buck.build/command/fix.html) fixes issues, and what options you have for more optimal fixes.

**Place annotations and constants in their own rules:** All annotations and compile-time constants (including enum values) used in the interface of a rule must be present during source-only ABI generation.

The error will suggest adding `required_for_source_abi = True` to any rule that defines an annotation type or a compile-time constant that is used from another rule, and [`buck fix`](https://buck.build/command/fix.html) will make that change. For best results, such rules should be manually inspected to ensure they are as small as possible—ideally containing only annotations, enums, and constants—and have as few dependencies as possible. In general, put annotations, enums, and constants in their own packages. This keeps to a minimum the amount of code that needs to be built before ABI generation. And because constants, enums, and annotations don't change as frequently as other kinds of code, those packages won't need to be rebuilt as frequently.

**Name packages and classes according to Java conventions:** Any packages that will not be available during source-only ABI generation must have names that begin with a lowercase letter. Any top-level classes that will not be available must have names that begin with an uppercase letter. These naming conventions are extremely common in Java.
The error will suggest renaming the package or class, but *you will have to make this change manually*; [`buck fix`](https://buck.build/command/fix.html) will not do it for you. If the package or class name in question cannot be changed, add `required_for_source_abi = True` to the build rule that defines it.

**Reference member types canonically:** Any references to member types that will not be available during source-only ABI generation must be canonical. That is, although member types are inherited by subclasses, a canonical reference refers to the member type as a member of the class in which it is defined, not one that inherits it.

If the defining class is accessible, the error will suggest using the canonical name. Otherwise, it will suggest adding `source_only_abi_deps` and [`buck fix`](https://buck.build/command/fix.html) will make whichever change is suggested by the error. For best results, consider making the defining class accessible so that the extra dependencies are not needed.

**Reference superclass member types unambiguously:** Any references to member types that are inherited by the current class and will not be available during source-only ABI generation must be at least partially qualified.
The error will suggest adding an import and partially qualifying the name, and [`buck fix`](https://buck.build/command/fix.html) will make this change.

**Provide enough deps for the compiler to run:** Source-only ABI generation involves running the Java compiler front-end over a rule's code without providing most of that rule's dependencies, then working with the compiler's data model to generate the ABI. The compiler is remarkably resilient to missing dependencies, but in some cases a missing dependency will prevent it from getting far enough for Buck to generate the source-only ABI. These cases typically involve inheritance, where a type is available during source-only ABI generation, but one of its superclasses is defined in another rule that is not available.

The error will suggest adding `source_only_abi_deps` to the rule so that the necessary dependencies will be available, and [`buck fix`](https://buck.build/command/fix.html) will make this change. For best results, limit the depth of class hierarchies—which is a best practice for Java design in many cases—and avoid splitting the class hierarchies across modules.

**Only single-type imports:** On-demand imports (star imports) and `static` imports of types cannot be resolved at source-ABI generation time unless the rule containing the type in question is available.

The error will suggest either adding a single-type import (in the case of star imports) or adding `source_only_abi_deps` in the case of `static` imports, and [`buck fix`](https://buck.build/command/fix.html) will make that change. For best results, consider changing `static` imports of types to non-static imports to avoid needing to add the extra dependencies.
