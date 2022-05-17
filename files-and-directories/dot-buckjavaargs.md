# .buckjavaargs

The root of your project may contain a configuration file named `.buckjavaargs`. If present, Buck will read this file and append any flags specified in it when launching its java process. Note the flags are only used when launching the main Buck java process, and not any other java tools Buck will invoke. The content of this file is split into individual arguments according to the rules of Python's `shlex.split`. On POSIX systems, Bourne shell quoting rules apply.
Here are some examples of why you would want to use `.buckjavaargs`.
To specify a larger heap size:

```
-Xmx2g
```

To ensure you can talk to the Maven Central Repo on a machine that uses IPv6:

```
-Djava.net.preferIPv6Addresses=true
```

