# Exopackage:
A technique for fast, iterative Android development.

Buck has an advanced feature to speed up iterative Android development called exopackage. An *exopackage* is a small shell of an Android app that contains the minimal code and resources needed to bootstrap loading the code for a full-fledged Android application. Loading the application code at runtime avoids a full reinstall of the app when testing a Java change, which dramatically reduces the length of edit/refresh cycles. Here are the performance improvements in build times for Buck vs. Gradle in a real world Android application, [AntennaPod](https://github.com/danieloeh/AntennaPod):

| 	|Gradle	|Buck	|Speed Up	|
|---	|---	|---	|---	|
|clean build	|31s	|6s	|5x	|
|---	|---	|---	|---	|
|incremental build	|13s	|1.7s	|7.5x	|
|no-op build	|3s	|0.2s	|15x	|
|clean install	|7.2s	|7.2s	|1x	|
|incremental install	|7.2s	|1.5s	|4.8x	|

(Note: These measurements were done on a MacBook Pro with a i7-3740QM CPU, with HyperThreading enabled, using Oracle Java 1.7.0_45 for Linux. We used 8 threads by running "./gradlew —parallel-threads 8". Gradle's daemon, parallel, and configureondemand options were enabled, as was Buck's daemon (which is enabled by default). In all cases, the builds were run multiple times to allow Java's JIT to warm up fully. The incremental build was adding a single blank line to a Java file.)
As you might expect, using exopackage requires you to make some code changes to your application. This article serves two purposes: it is both a **tutorial** that shows how to migrate an Android app that builds with Gradle over to Buck with exopackage, as well as a **technical explanation** of how exopackage works.
For this tutorial, we will demonstrate how to use exopackage by adding Buck support to [AntennaPod](https://github.com/danieloeh/AntennaPod), an open source podcast management app for Android. Each step in the process is documented as a separate commit in [our fork of AntennaPod](https://github.com/facebookarchive/AntennaPod). Note that most of the work in this tutorial is in adding Buck support to AntennaPod. If your Android project already uses Buck, then you can jump straight to [Step 5](https://buck.build/article/exopackage.html#build-buck-support-library), which will require minimal changes to your existing project.

**Table of Contents**

* Step 1: Check out AntennaPod
* Step 2: Import JARs for third party dependencies
* Step 3: Ensure R.* constants are not assumed to be final
* Step 4: Create BUCK files that define build rules to build AntennaPod with Buck
* Step 5: Build Buck's Android support library
* Step 6: Modify AntennaPod to use exopackage
* Step 7: Profit!
* Caveats
* Incompatible Devices



## Step 1: Check out AntennaPod

If you want to walk through this tutorial and make all of the changes yourself, then the first step is to clone the AntennaPod application at the same revision used to create this tutorial:

```
git clone --recursive git@github.com:facebookarchive/AntennaPod.git
cd AntennaPod
git checkout c2080b1dfd17fc371e04ce1e7b39ebadaf3cb7a7
```

If you just want to play with the final version of the tutorial after all of the Buck/exopackage support has been added, then checkout to the appropriate revision so you can build and run AntennaPod using Buck. Note that you must add your own keystore before you can do a build. (We do not check in `debug.keystore` for security reasons.)

```
git checkout 4250292b4d4742d40a9b06ce638741b2873210f1
cp ~/.android/debug.keystore keystore/debug.keystore
buck install --run antennapod
```

## Step 2: Import JARs for third party dependencies

View on GitHub: [6c809273e70428f2f465cdeef568e1f35c30b439](https://github.com/facebookarchive/AntennaPod/commit/6c809273e70428f2f465cdeef568e1f35c30b439)
Unlike Gradle, Buck requires that all files that contribute to the project live under the project root (which is defined by the presence of a `.buckconfig` file). Instead of downloading third party JARs from the Maven Central Repository as part of the build process (like Gradle), Buck expects such dependencies to live in version control, just like application code. This ensures that builds are *reproducible* and *hermetic*.
For AntennaPod, we ran `./gradlew --debug assembleDebug` and inspected the output to figure out which third party JAR files Gradle was using to build the app. As a result, we ended up adding the following files to the `libs` directory, which also includes an [AAR](https://developer.android.com/studio/projects/android-library#aar-contents) file for the Android support library for v7 compatibility.

```
libs/appcompat-v7-19.1.0.aar
libs/commons-io-2.4.jar
libs/commons-lang3-3.3.2.jar
libs/flattr4j-core-2.10.jar
libs/library-2.4.0.jar
libs/support-v4-19.1.0.jar
```

Note that we also removed the `libs` directory from `.gitignore` as part of this change.

## Step 3: Ensure R.* constants are not assumed to be final

View on GitHub: [5f7ce4934e9e8ea9a391a09d2093c79b64b7d207](https://github.com/facebookarchive/AntennaPod/commit/5f7ce4934e9e8ea9a391a09d2093c79b64b7d207)
If you have any code like the following:

```
  int id = view.getId();
  switch (id) {
    case R.id.button1:
      action1();
      break;
    case R.id.button2:
      action2();
      break;
    case R.id.button3:
      action3();
      break;
  }
```

You must convert it to use `if`/`else` blocks as follows:

```
  int id = view.getId();
  if (id == R.id.button1) {
    action1();
  } else if (id == R.id.button2) {
    action2();
  } else if (id == R.id.button3) {
    action3();
  }
```

As explained in the article [Non-constant Fields in Case Labels](http://tools.android.com/tips/non-constant-fields), the constants in the `R` class for an Android library projects are not `final`, which means they cannot be used as constant expressions in `case` statements. Because Buck treats the code for an [`android_library`](https://buck.build/rule/android_library.html) as if it were part of an Android library project, this applies to all Android code built by Buck. The article explains how you can leverage your IDE to automate this refactoring.

## Step 4: Create BUCK files that define build rules to build AntennaPod with Buck

In Buck, build rules are defined in build files named `BUCK`. In this step, we create a `BUCK` file and add the build rules necessary to build the AntennaPod APK using Buck without touching any other files in the AntennaPod repository.

We start by creating a `BUCK` file and defining an [`android_library`](https://buck.build/rule/android_library.html) rule that exposes all of the JARs in the `libs` directory as a single dependency, `:all-jars`:

```
  import re

  jar_deps = []
  for jarfile in glob(['libs/*.jar']):
    name = 'jars__' + re.sub(r'^.*/([^/]+)\.jar$', r'\1', jarfile)
    jar_deps.append(':' + name)
    prebuilt_jar(
      name = name,
      binary_jar = jarfile,
    )

  android_library(
    name = 'all-jars',
    exported_deps = jar_deps,
  )
```

We also wrap the AAR file for the Android support library with an [`android_prebuilt_aar`](https://buck.build/rule/android_prebuilt_aar.html) rule:

```
  android_prebuilt_aar(
    name = 'appcompat',
    aar = 'libs/appcompat-v7-19.1.0.aar',)
```

Next, we define some rules to generate `.java` files from `.aidl` files and package them as an [`android_library`](https://buck.build/rule/android_library.html), as well:

```
  presto_gen_aidls = []
  for aidlfile in glob(['src/com/aocate/presto/service/*.aidl']):
    name = 'presto_aidls__' + re.sub(r'^.*/([^/]+)\.aidl$', r'\1', aidlfile)
    presto_gen_aidls.append(':' + name)
    gen_aidl(
      name = name,
      aidl = aidlfile,
      import_path = 'src',
    )

  android_library(
    name = 'presto-aidls',
    srcs = presto_gen_aidls,
  )
```


Then we define an [`android_build_config`](https://buck.build/rule/android_build_config.html), which will generate `de.danoeh.antennapod.BuildConfig` for us, compile it, and expose it as a [`java_library`](https://buck.build/rule/java_library.html). As we will see, this class plays an important role in creating an exopackage:


```
  android_build_config(
    name = 'build-config',
    package = 'de.danoeh.antennapod',)
```

Before we can define an [`android_library`](https://buck.build/rule/android_library.html) rule to compile the primary sources for AntennaPod, we must define some rules to bundle the resources and code for its dependent Android library projects:

```
  android_resource(
    name = 'dslv-res',
    package = 'com.mobeta.android.dslv',
    res = 'submodules/dslv/library/res',
  )

  android_library(
    name = 'dslv-lib',
    srcs = glob(['submodules/dslv/library/src/**/*.java']),
    deps = [
      ':all-jars',
      ':dslv-res',
    ],
  )

  android_library(
    name = 'presto-lib',
    srcs = glob(['src/com/aocate/**/*.java']),
    deps = [
      ':all-jars',
      ':presto-aidls',
    ],
  )
```


Now that the dependent Android library projects can be expressed as dependencies in Buck, we define [`android_resource`](https://buck.build/rule/android_resource.html) and [`android_library`](https://buck.build/rule/android_library.html) rules that build the main AntennaPod code:


```
  android_resource(
    name = 'res',
    package = 'de.danoeh.antennapod',
    res = 'res',
    assets = 'assets',
    deps = [':appcompat',':dslv-res',])

  android_library(
    name = 'main-lib',
    srcs = glob(['src/de/**/*.java']),
    deps = [':all-jars',':appcompat',':build-config',':dslv-lib',':presto-lib',':res',],)
```

To package the Android code into an APK, we need a keystore with which it should be signed, a manifest that defines the app, and a rule to package everything toegether. Let's start with the keystore, as defining this rule requires an extra step from the command line:

```
keystore(
  name = 'debug_keystore',
  store = 'keystore/debug.keystore',
  properties = 'keystore/debug.keystore.properties',)
```

Note that a clean checkout of the AntennaPod repository includes a `keystore/debug.keystore.properties` file, but no `keystore/debug.keystore` file. This is because the Android Developer Tools creates a keystore with a common set of credentials under `~/.android/debug.keystore` on your machine. Assuming you have not changed this default, the values in `keystore/debug.keystore.properties` will be appropriate for your `~/.android/debug.keystore`. Recall that Buck requires all files it must know about to live under the project root, so **you must copy the keystore to your project where Buck expects it**:

```
cp ~/.android/debug.keystore keystore/debug.keystore
```

With the [`keystore`](https://buck.build/rule/keystore.html) defined, now we can define the [`android_binary`](https://buck.build/rule/android_binary.html) rule whose output will be the AntennaPod APK. Note that the only item listed in its `deps` is `:main-lib`, as [`android_binary`](https://buck.build/rule/android_binary.html) will package `:main-lib` and its transitive dependencies into the APK.

```
android_binary(
  name = 'antennapod',
  manifest = 'AndroidManifest.xml',
  keystore = ':debug_keystore',
  deps = [
    ':main-lib',
  ],
)
```

To facilitate building from the command line (and to leverage the build cache), create a file named `.buckconfig` in the root of the repo with the following contents:

```
[alias]
  antennapod = //:antennapod
[cache]
  mode = dir
  dir_max_size = 1GB
[android]
  target = Google Inc.:Google APIs:19
```

Now you should be able to run [`buck build`](https://buck.build/command/build.html)` antennapod` to build the app, or [`buck install`](https://buck.build/command/install.html)` antennapod` to install it if `adb devices` is not empty.

## Step 5: Build Buck's Android support library

View on GitHub: [1f1b375624664ac1cdc82668a4b952c6f332bdb0](https://github.com/facebookarchive/AntennaPod/commit/1f1b375624664ac1cdc82668a4b952c6f332bdb0)
In order for your app to use exopackage, it needs to use Buck's Java library that provides support for it. You can easily build this library from source from your checkout of Buck as follows:

```
**# Run this from the root of your checkout of Buck, not from AntennaPod.**
buck build --out buck-android-support.jar buck-android-support
```

Once you have built it, move it over to AntennaPod's `libs` directory, just like the other third party JAR files:

```
mv buck-android-support.jar path/to/AntennaPod/libs
```

## Step 6: Modify AntennaPod to use exopackage

View on GitHub: [4250292b4d4742d40a9b06ce638741b2873210f1](https://github.com/facebookarchive/AntennaPod/commit/4250292b4d4742d40a9b06ce638741b2873210f1)

On a high level, the main thing that you need to do to leverage exopackage is change the insertion point of your app from the existing `android.app.Application` that your app uses to an [`ExopackageApplication`](http://buck.build/javadoc/com/facebook/buck/android/support/exopackage/ExopackageApplication.html) that delegates to your original `Application`. This level of indirection is what makes it possible for exopackage to dynamically load the code for your application in debug mode. In release mode, [`ExopackageApplication`](http://buck.build/javadoc/com/facebook/buck/android/support/exopackage/ExopackageApplication.html) expects all of the code for your app to be present in the APK, so it skips the step where it tries to dynamically load code.

If your app has a class that subclasses `android.app.Application` that is listed as the main app in `AndroidManifest.xml` via the `<application name>` attribute, then the first thing that you need to do is modify that class so it extends [`DefaultApplicationLike`](http://buck.build/javadoc/com/facebook/buck/android/support/exopackage/DefaultApplicationLike.html) rather than `Application`:

```
-public class PodcastApp extends Application {
+public class PodcastApp extends DefaultApplicationLike {
```

Further, your [`DefaultApplicationLike`](http://buck.build/javadoc/com/facebook/buck/android/support/exopackage/DefaultApplicationLike.html) must declare a constructor that takes an `Application` as its only parameter. You most likely want to store it as a field:


```
  private final Application appContext;

  public PodcastApp(Application appContext) {
    this.appContext = appContext;
  }
```

Now all methods that previously accessed the API of `Application` via inheritance can delegate to the `appContext` instance instead:

```
-LOGICAL_DENSITY = getResources().getDisplayMetrics().density;
+LOGICAL_DENSITY = appContext.getResources().getDisplayMetrics().density;
```

Now you must create your new `Application` class, which will be a subclass of [`ExopackageApplication`](http://buck.build/javadoc/com/facebook/buck/android/support/exopackage/ExopackageApplication.html). As you can see from its API, it is an `abstract` class that does not have a default constructor, so you must define a no-arg constructor as follows:

```
  package de.danoeh.antennapod;

  import com.facebook.buck.android.support.exopackage.ExopackageApplication;

  public class AppShell extends ExopackageApplication {

    public AppShell() {
      super(
        // This is passed as a string so the shell application does not
        // have a binary dependency on your ApplicationLike class.
        /* applicationLikeClassName */ "de.danoeh.antennapod.PodcastApp",

        // The package for this BuildConfig class must match the package
        // from the android_build_config() rule. The value of the flags
        // will be set based on the "exopackage_modes" argument to
        // android_binary().
        de.danoeh.antennapod.BuildConfig.EXOPACKAGE_FLAGS);
    }
  }
```

Alternatively, if your original app did not have a custom subclass of `android.app.Application`, then you do not have to create an implementation of `ApplicationLike`. You must still create a subclass of `ExopackageApplication`, but now your implementation can be simpler:

```
  package de.danoeh.antennapod;

  import com.facebook.buck.android.support.exopackage.ExopackageApplication;

  public class AppShell extends ExopackageApplication {

    public AppShell() {
      super(de.danoeh.antennapod.BuildConfig.EXOPACKAGE_FLAGS);
    }
  }
```

Now the more sophisticated changes will be in the `BUCK` file where you defined your [`android_binary`](https://buck.build/rule/android_binary.html) rule. First, you will need to create an [`android_library`](https://buck.build/rule/android_library.html) that builds your [`ExopackageApplication`](http://buck.build/javadoc/com/facebook/buck/android/support/exopackage/ExopackageApplication.html):

```
  APP_CLASS_SOURCE = 'src/de/danoeh/antennapod/AppShell.java'

  android_library(
    name = 'application-lib',
    srcs = [APP_CLASS_SOURCE],
    deps = [
      # This is the android_build_config() rule that you created in Step 4.
      # If you jumped straight to Step 5 because your Android app was already
      # configured to build with Buck, then go back to Step 4 and add this rule
      # if you aren't already using an android_build_config().
      ':build-config',

      # This is the prebuilt_jar() rule that wraps buck-android-support.jar.
      ':jars__buck-android-support',
    ],
  )
```


If you have an existing [`android_library`](https://buck.build/rule/android_library.html) rule that [`glob()`](https://buck.build/function/glob.html)s your [`ExopackageApplication`](http://buck.build/javadoc/com/facebook/buck/android/support/exopackage/ExopackageApplication.html)'s source file, then make sure to exclude it:

```
-  srcs = glob(['src/de/**/*.java']),
+  srcs = glob(['src/de/**/*.java'], excludes = [APP_CLASS_SOURCE]),
```

The biggest change to your `BUCK` file will be the new arguments to your [`android_binary`](https://buck.build/rule/android_binary.html) rule (new lines are highlighted in green):

```
android_binary(
  name = 'antennapod',
  manifest = 'AndroidManifest.xml',
  keystore = ':debug_keystore',  use_split_dex = True,
  exopackage_modes = ['secondary_dex'],
  primary_dex_patterns = [
    '^de/danoeh/antennapod/AppShell^',
    '^de/danoeh/antennapod/BuildConfig^',
    '^com/facebook/buck/android/support/exopackage/',
  ],  
  deps = [    ':application-lib',
    ':main-lib',
  ],
)
```

As you might have guessed, `primary_dex_patterns` is a pattern that identifies which `.class` files from the transitive `deps` that must be included in the shell app that bootstraps the rest of the app. As such, these patterns match the transitive deps of `:application-lib`.

Setting `exopackage_modes = ['secondary_dex']` is what ensures that `BuildConfig.EXOPACKAGE_FLAGS` will be set correctly, in addition to the other packaging changes that Buck makes to support exopackage. This must used with `use_split_dex = True` because using exopackage requires dividing the app into multiple dex files.

Finally, you must update your `AndroidManifest.xml` to refer to the `ExopackageApplication` as the new entry point into your app:

```
-android:name="de.danoeh.antennapod.PodcastApp"
+android:name="de.danoeh.antennapod.AppShell"
```

## Step 7: Profit!

Now your development cycle should be as follows:

```
buck install --run antennapod
# Edit your application's Java code.
buck install --run antennapod
# Watch in amazement as your changes are loaded faster than ever before!
```

## Caveats

Currently, exopackage speeds up incremental install times for Java changes, but changes to Android resources or native libraries require a full reinstall. This is something we hope to improve in the future.
Be aware of the following limitations when using Buck and exopackage:

* You cannot use `adb install` for exopackages. You must use [`buck install`](https://buck.build/command/install.html).
* You should use [`buck uninstall`](https://buck.build/command/uninstall.html) instead of `adb uninstall` to uninstall the app. Otherwise, unnecessary files will be left in `/data/local/tmp`. You can remove them with `adb shell rm -r /data/local/tmp/exopackage/$PACKAGE_NAME`.
* Some devices are not compatible with the exopackage installer. [See below](https://buck.build/article/exopackage.html#incompatible-devices).
* Install to SD card does not work right now.
* Exopackages will not start up for non-primary users on a multi-user android device.
* When you do an install, system notifications and alarms will **not** be cleared, so you might get an intent back from them with the old version of your `Parcelable`, which could cause a crash or other confusing behavior.
* When you do an install on pre-ICS devices, the app will not be stopped.

## Incompatible Devices

Empirically, we have determined that the following devices do not work with exopackage:

* Some AOSP builds between the KitKat release and L Preview.

You might want to keep two versions of your [`android_binary`](https://buck.build/rule/android_binary.html) rule in your `BUCK` files: one that uses exopackage and one that does not. That way, you will always have a way to test on devices that do not support exopackage.
