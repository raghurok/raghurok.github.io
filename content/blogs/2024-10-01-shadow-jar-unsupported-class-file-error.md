+++
title = 'Shadow Jar Unsupported Class File Error'
categories = ['build']
date = 2024-10-01T00:00:00-07:00
draft = false
tags = ['shadowJar', 'gradle']
+++
When upgrading `com.fasterxml.jackson` from `v2.13.1` to `v2.17.2`, I started encountering the 
`Unsupported class file major version 61` error when building `shadowJar` with the
`com.github.johnrengelman.shadow:6.1.0` plugin.

The `Unsupported class file major version 61` error indicates that somewhere in the classpath, shadowJar encountered
a class compiled with Java 17, while the current runtime only supports up to Java 11. The Gradle stacktrace
does not specify which class failed. The stacktrace was as follows:
```java
ex
java.lang.IllegalArgumentException: Unsupported class file major version 61
        at shadow.org.objectweb.asm.ClassReader.<init>(ClassReader.java:199)
        at shadow.org.objectweb.asm.ClassReader.<init>(ClassReader.java:180)
        at shadow.org.objectweb.asm.ClassReader.<init>(ClassReader.java:166)
        at shadow.org.objectweb.asm.ClassReader.<init>(ClassReader.java:287)
        at jdk.internal.reflect.GeneratedConstructorAccessor509.newInstance(Unknown Source)
        at java.base/jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.base/java.lang.reflect.Constructor.newInstance(Constructor.java:490)
        at org.codehaus.groovy.reflection.CachedConstructor.invoke(CachedConstructor.java:72)
        at org.codehaus.groovy.runtime.callsite.ConstructorSite$ConstructorSiteNoUnwrapNoCoerce.callConstructor(ConstructorSite.java:105)
        ...
```
To try to debug, I ran the Gradle task with debug enabled using `-Dorg.gradle.debug=true --no-daemon` and
connected IntelliJ IDEA (IJ) remote debug to port 5005. I also added breakpoint in `IllegalArgumentException` class. The build did
hit the breakpoint. Using IJ's evaluate expression, I serialized the `buf` field in `ClassReader`
(`bytes[]` field that holds the class being relocated) to a file in the filesystem. Finally, using `javap`
to inspect the class showed that the failure occurred when relocating the `FastDoubleSwar` class from the jackson-core library.
```
javap -verbose <saved-file>
...
class com.fasterxml.jackson.core.io.doubleparser.FastDoubleSwar
minor version: 0
major version: 61
```
However, the jackson-core does support JDK 8, and when I inspected the Gradle cached jar, it indicated that 
the class was compiled with Java 8.
```sh
javap -verbose -cp ~/.gradle/caches/modules-2/files-2.1/com.fasterxml.jackson.core/jackson-core/2.17.2/969a35cb35c86512acbadcdbbbfb044c877db814/jackson-core-2.17.2.jar com.fasterxml.jackson.core.io.doubleparser.FastDoubleSwar | grep "major"
  major version: 52
```
Further investigation led me to the concept of [multi-release JARs](https://openjdk.org/jeps/238) (MRJARs).
Introduced in Java 9, MRJARs allow library maintainers to package optimized versions of classes for different Java
versions within a single JAR file. It turned out that jackson-core, starting from version 2.15.x, had started releasing MRJARs,
including Java 17 and Java 21-specific implementations of `FastDoubleSwar` and a few other classes. These classes are 
located in the `META-INF/versions` directory and when `shadowJar` reads the `META-INF` directory it fails with the error. There is 
no option to exclude reading this folder.

The Shadow JAR plugin, in its older version, was not equipped to handle these multi-release JARs, 
it's a known [issue](https://github.com/GradleUp/shadow/issues/878). The fix is to upgrade the plugin 
to the latest version. The `v8.3.0` version of `shadow-jar` plugin upgrades the `ASM` library to `9.7`, 
which supports Java version up to 21. Since the `jackson` MRJAR includes class files up to version 21  , 
the plugin must be upgraded to at least v8.3.0.

