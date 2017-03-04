**Note: the latest Android Gradle plugin has now built in support for annotation processors and actively blocks `android-apt`, see [this wiki page on how to migrate](https://bitbucket.org/hvisser/android-apt/wiki/Migration) and [this blogpost](http://www.littlerobots.nl/blog/Whats-next-for-android-apt/) for more info.** 

What is this?
---------------
The android-apt plugin assists in working with annotation processors in combination with Android Studio. It has two purposes:

* Allow to configure a compile time only annotation processor as a dependency, not including the artifact in the final APK or library
* Set up the source paths so that code that is generated from the annotation processor is correctly picked up by Android Studio.

This plugin requires the `android` or `android-library` plugin (version 0.9.x or up) to be configured on your project.

Including and using the plugin in your build script
---------------------------------------------------
Add the following to your build script to use the plugin:
```
#!groovy
buildscript {
    repositories {
      mavenCentral()
    }
    dependencies {
        // replace with the current version of the Android plugin
        classpath 'com.android.tools.build:gradle:1.3.0'
        // the latest version of the android-apt plugin
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
}

apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt'
```

Passing processor arguments
---------------------------
Some annotation processor may require to pass custom arguments, you can use `apt.arguments` for that purpose.
For instance for AndroidAnnotations you can use the following configuration:

```
#!groovy
apt {
    arguments {
            resourcePackageName android.defaultConfig.applicationId
            androidManifestFile variant.outputs[0]?.processResources?.manifestFile
    }
}
```

AndroidAnnotations needs to know where your `AndroidManifest.xml` is. Retrieving it through the `variant` allows for different `AndroidManfest.xml` files for each flavor. However, not all variants (e.g. unit tests) might have the properties required, hence the groovy `?.` operator to dereference these properties in a safe way. (Yes, this is a bit of a hack).

The arguments are processed for each variant when the compiler is configured. From this closure you can reference `android`, `project` and `variant` for the current variant.
Because this is just a Groovy closure things like setting configuration based on the `variant` name can also be done:
 
```
#!groovy

def getManifestByVariant(variant) {
    // return the value based on the variant
    if (variant.name == 'release') {
        return '/my/path/to/the/manifest.xml'
    }
    return variant.outputs[0]?.processResources?.manifestFile
}

apt {
     arguments {
         if (variant.name == 'debug') {
            resourcePackageName "com.myapp.package.name.debug"
            // more options
         }             
         androidManifestFile project.getManifestByVariant(variant)             
     }
}
```

Configurating a compiler style dependency
-----------------------------------------
Annotation processors generally have a API and a processor that generates code that is used by the API. Depending on the project the processor and the API might be split up in separate dependencies. For example, [Dagger][1] uses two artifacts called _dagger-compiler_ and _dagger_. The compiler artifact is only used during compilation, while the main _dagger_ artifact is required at runtime.

To aid in configuring this dependency, the plugin adds a Gradle [configuration][2] named **apt** that can be used like this:

```
#!groovy
dependencies {
 apt 'com.squareup.dagger:dagger-compiler:1.1.0'
 compile 'com.squareup.dagger:dagger:1.1.0'
}
```

If your instrumentation test code requires generated code to be visible in Android Studio, you can use the `androidTestApt` configuration:

```
#!groovy
dependencies {
 androidTestApt 'com.github.frankiesardo:android-auto-value-processor:0.1'
 androidTestCompile 'com.github.frankiesardo:android-auto-value:0.1'
}
```

For unit tests use `testApt` **Note: requires Android Studio 1.4!**

```
#!groovy
dependencies {
 testApt 'com.github.frankiesardo:android-auto-value-processor:0.1'
 testCompile 'com.github.frankiesardo:android-auto-value:0.1'
}
```


Additional configuration
------------------------
It's possible to specify the class names of processors that need to be run during compilation and in that case disable
the default auto discovery mechanism of `javac` as well, by specifying the classes in the `apt` block:

```
#!groovy
apt {
    processor "my.class.name"
    processor "another.processor.class.name"
    // if you only want to run the processors above
    disableDiscovery true
}
```

Configuration of other annotation processors
--------------------------------------------
For annotation processors that include the API and processor in one artifact, there's no special setup. You just add the artifact to the _compile_ configuration like usual and everything will work like normal. Additionally, if code that is generated by the processor is to be referenced in your own code, Android Studio will now correctly reference that code and resolve references to it.

[1]:http://square.github.io/dagger
[2]:http://www.gradle.org/docs/current/userguide/artifact_dependencies_tutorial.html

FAQ
---
Q: When do I need this plugin?

A: You _need_ this plugin when you include processors that generate sources that you need to import in your code. For example when using
[Dagger 2][9] or [Android Annotations][10]. Using this plugin Android Studio will be configured to place the generated sources on the
build path, preventing errors in the IDE. When using the `apt` scope, the annotation processor will not be included in your final APK.

Q: What's the difference in using `provided` vs using `apt` for any annotation processor?

A: `provided` will put the annotation processor classes and it's dependencies on the IDE class path. This means that you could
accidentally reference these classes. For example, if the annotation processor uses Guava, you might mistakenly import that in
your Android code. At runtime this would cause a crash. When using `apt` the processor is not added to your class path; it's only used
for annotation processing. The plugin will also make sure that Android Studio is configured to put any generated source on the IDE build path
so that you can reference those in Android Studio, if needed.

History & Credits
---------------
This plugin is based on a [script][6] that I've been using for some time which is the result of [this post on Google+][7] and [this post on StackOverflow.com][8].
Variations of the SO post and my gists have been floating around for a while on the interwebs. That, and the fact that including scripts is a bit inconvenient pushed me to create this plugin.

License
-------
This plugin is created by Hugo Visser and released in the [public domain][3]. Feel free to use and adapt as you like.
To get in touch, hit me up on [Twitter][4] or [Google Plus][5].

[3]:http://unlicense.org/
[4]:https://twitter.com/botteaap
[5]:https://google.com/+hugovisser
[6]:https://bitbucket.org/qbusict/android-gradle-scripts/src/686ce2301245ab1f0e6a32fb20b4d246ef742223/annotations.groovy?at=default
[7]:https://plus.google.com/+HugoVisser/posts/VtGYV8RHwmo
[8]:http://stackoverflow.com/questions/16683944/androidannotations-nothing-generated-empty-activity
[9]:http://google.github.io/dagger/
[10]:http://androidannotations.org/