Chapter 7. The Gradle Daemon

Table of Contents

7.1. Why the Gradle Daemon is important for performance
7.2. Running Daemon Status
7.3. Disabling the Daemon
7.4. Stopping an existing Daemon
7.5. FAQ
7.6. Tools & IDEs
7.7. How does the Gradle Daemon make builds faster?

From Wikipedia…

    A daemon is a computer program that runs as a background process, rather than being under the direct control of an interactive user.

Gradle runs on the Java Virtual Machine (JVM) and uses several supporting libraries that require a non-trivial initialization time. As a result, it can sometimes seem a little slow to start. The solution to this problem is the Gradle Daemon: a long-lived background process that executes your builds much more quickly than would otherwise be the case. We accomplish this by avoiding the expensive bootstrapping process as well as leveraging caching, by keeping data about your project in memory. Running Gradle builds with the Daemon is no different than without. Simply configure whether you want to use it or not - everything else is handled transparently by Gradle.
7.1. Why the Gradle Daemon is important for performance

The Daemon is a long-lived process, so not only are we able to avoid the cost of JVM startup for every build, but we are able to cache information about project structure, files, tasks, and more in memory.

The reasoning is simple: improve build speed by reusing computations from previous builds. However, the benefits are dramatic: we typically measure build times reduced by 15-75% on subsequent builds. We recommend profiling your build by using --profile to get a sense of how much impact the Gradle Daemon can have for you.

The Gradle Daemon is enabled by default starting with Gradle 3.0, so you don’t have to do anything to benefit from it.

If you run CI builds in ephemeral environments (such as containers) that do not reuse any processes, use of the Daemon will slightly decrease performance (due to caching additional information) for no benefit, and may be disabled.
7.2. Running Daemon Status

To get a list of running Gradle Daemons and their statuses use the --status command.

Sample output:

  PID VERSION                 STATUS
28411 3.0                     IDLE
34247 3.0                     BUSY

Currently, a given Gradle version can only connect to daemons of the same version. This means the status output will only show Daemons for the version of Gradle being invoked and not for any other versions. Future versions of Gradle will lift this constraint and will show the running Daemons for all versions of Gradle.
7.3. Disabling the Daemon

The Gradle Daemon is enabled by default, and we recommend always enabling it. There are several ways to disable the Daemon, but the most common one is to add the line

org.gradle.daemon=false

to the file «USER_HOME»/.gradle/gradle.properties, where «USER_HOME» is your home directory. That’s typically one of the following, depending on your platform:

    C:\Users\<username> (Windows Vista & 7+)

    /Users/<username> (macOS)

    /home/<username> (Linux)

If that file doesn’t exist, just create it using a text editor. You can find details of other ways to disable (and enable) the Daemon in Section 7.5, “FAQ” further down. That section also contains more detailed information on how the Daemon works.

Note that having the Daemon enabled, all your builds will take advantage of the speed boost, regardless of the version of Gradle a particular build uses.
Continuous integration

Since Gradle 3.0, we enable Daemon by default and recommend using it for both developers' machines and Continuous Integration servers. However, if you suspect that Daemon makes your CI builds unstable, you can disable it to use a fresh runtime for each build since the runtime is completely isolated from any previous builds.
7.4. Stopping an existing Daemon

As mentioned, the Daemon is a background process. You needn’t worry about a build up of Gradle processes on your machine, though. Every Daemon monitors its memory usage compared to total system memory and will stop itself if idle when available system memory is low. If you want to explicitly stop running Daemon processes for any reason, just use the command gradle --stop.

This will terminate all Daemon processes that were started with the same version of Gradle used to execute the command. If you have the Java Development Kit (JDK) installed, you can easily verify that a Daemon has stopped by running the jps command. You’ll see any running Daemons listed with the name GradleDaemon.
7.5. FAQ
7.5.1. How do I disable the Gradle Daemon?

There are two recommended ways to disable the Daemon persistently for an environment:

    Via environment variables: add the flag -Dorg.gradle.daemon=false to the GRADLE_OPTS environment variable

    Via properties file: add org.gradle.daemon=false to the «GRADLE_USER_HOME»/gradle.properties file

Note, «GRADLE_USER_HOME» defaults to «USER_HOME»/.gradle, where «USER_HOME» is the home directory of the current user. This location can be configured via the -g and --gradle-user-home command line switches, as well as by the GRADLE_USER_HOME environment variable and org.gradle.user.home JVM system property.

Both approaches have the same effect. Which one to use is up to personal preference. Most Gradle users choose the second option and add the entry to the user gradle.properties file.

On Windows, this command will disable the Daemon for the current user:

(if not exist "%USERPROFILE%/.gradle" mkdir "%USERPROFILE%/.gradle") && (echo. >> "%USERPROFILE%/.gradle/gradle.properties" && echo org.gradle.daemon=false >> "%USERPROFILE%/.gradle/gradle.properties")

On UNIX-like operating systems, the following Bash shell command will disable the Daemon for the current user:

mkdir -p ~/.gradle && echo "org.gradle.daemon=false" >> ~/.gradle/gradle.properties

Once the Daemon is disabled for a build environment in this way, a Gradle Daemon will not be started unless explicitly requested using the --daemon option.

The --daemon and --no-daemon command line options enable and disable usage of the Daemon for individual build invocations when using the Gradle command line interface. These command line options have the highest precedence when considering the build environment. Typically, it is more convenient to enable the Daemon for an environment (e.g. a user account) so that all builds use the Daemon without requiring to remember to supply the --daemon option.
7.5.2. Why is there more than one Daemon process on my machine?

There are several reasons why Gradle will create a new Daemon, instead of using one that is already running. The basic rule is that Gradle will start a new Daemon if there are no existing idle or compatible Daemons available. Gradle will kill any Daemon that has been idle for 3 hours or more, so you don’t have to worry about cleaning them up manually.

idle

    An idle Daemon is one that is not currently executing a build or doing other useful work.
compatible

    A compatible Daemon is one that can (or can be made to) meet the requirements of the requested build environment. The Java runtime used to execute the build is an example aspect of the build environment. Another example is the set of JVM system properties required by the build runtime.

Some aspects of the requested build environment may not be met by an Daemon. If the Daemon is running with a Java 7 runtime, but the requested environment calls for Java 8, then the Daemon is not compatible and another must be started. Moreover, certain properties of a Java runtime cannot be changed once the JVM has started. For example, it is not possible to change the memory allocation (e.g. -Xmx1024m), default text encoding, default locale, etc of a running JVM.

The “requested build environment” is typically constructed implicitly from aspects of the build client’s (e.g. Gradle command line client, IDE etc.) environment and explicitly via command line switches and settings. See Chapter 12, The Build Environment for details on how to specify and control the build environment.

The following JVM system properties are effectively immutable. If the requested build environment requires any of these properties, with a different value than a Daemon’s JVM has for this property, the Daemon is not compatible.

    file.encoding

    user.language

    user.country

    user.variant

    java.io.tmpdir

    javax.net.ssl.keyStore

    javax.net.ssl.keyStorePassword

    javax.net.ssl.keyStoreType

    javax.net.ssl.trustStore

    javax.net.ssl.trustStorePassword

    javax.net.ssl.trustStoreType

    com.sun.management.jmxremote

The following JVM attributes, controlled by startup arguments, are also effectively immutable. The corresponding attributes of the requested build environment and the Daemon’s environment must match exactly in order for a Daemon to be compatible.

    The maximum heap size (i.e. the -Xmx JVM argument)

    The minimum heap size (i.e. the -Xms JVM argument)

    The boot classpath (i.e. the -Xbootclasspath argument)

    The “assertion” status (i.e. the -ea argument)

The required Gradle version is another aspect of the requested build environment. Daemon processes are coupled to a specific Gradle runtime. Working on multiple Gradle projects during a session that use different Gradle versions is a common reason for having more than one running Daemon process.
7.5.3. How much memory does the Daemon use and can I give it more?

If the requested build environment does not specify a maximum heap size, the Daemon will use up to 1GB of heap. It will use the JVM’s default minimum heap size. 1GB is more than enough for most builds. Larger builds with hundreds of subprojects, lots of configuration, and source code may require, or perform better, with more memory.

To increase the amount of memory the Daemon can use, specify the appropriate flags as part of the requested build environment. Please see Chapter 12, The Build Environment for details.
7.5.4. How can I stop a Daemon?

Daemon processes will automatically terminate themselves after 3 hours of inactivity or less. If you wish to stop a Daemon process before this, you can either kill the process via your operating system or run the gradle --stop command. The --stop switch causes Gradle to request that all running Daemon processes, of the same Gradle version used to run the command, terminate themselves.
7.5.5. What can go wrong with Daemon?

Considerable engineering effort has gone into making the Daemon robust, transparent and unobtrusive during day to day development. However, Daemon processes can occasionally be corrupted or exhausted. A Gradle build executes arbitrary code from multiple sources. While Gradle itself is designed for and heavily tested with the Daemon, user build scripts and third party plugins can destabilize the Daemon process through defects such as memory leaks or global state corruption.

It is also possible to destabilize the Daemon (and build environment in general) by running builds that do not release resources correctly. This is a particularly poignant problem when using Microsoft Windows as it is less forgiving of programs that fail to close files after reading or writing.

Gradle actively monitors heap usage and attempts to detect when a leak is starting to exhaust the available heap space in the daemon. When it detects a problem, the Gradle daemon will finish the currently running build and proactively restart the daemon on the next build. This monitoring is enabled by default, but can be disabled by setting the org.gradle.daemon.performance.enable-monitoring system property to false.

If it is suspected that the Daemon process has become unstable, it can simply be killed. Recall that the --no-daemon switch can be specified for a build to prevent use of the Daemon. This can be useful to diagnose whether or not the Daemon is actually the culprit of a problem.
7.6. Tools & IDEs

The Gradle Tooling API (see Chapter 14, Embedding Gradle using the Tooling API), that is used by IDEs and other tools to integrate with Gradle, always use the Gradle Daemon to execute builds. If you are executing Gradle builds from within you’re IDE you are using the Gradle Daemon and do not need to enable it for your environment.
7.7. How does the Gradle Daemon make builds faster?

The Gradle Daemon is a long lived build process. In between builds it waits idly for the next build. This has the obvious benefit of only requiring Gradle to be loaded into memory once for multiple builds, as opposed to once for each build. This in itself is a significant performance optimization, but that’s not where it stops.

A significant part of the story for modern JVM performance is runtime code optimization. For example, HotSpot (the JVM implementation provided by Oracle and used as the basis of OpenJDK) applies optimization to code while it is running. The optimization is progressive and not instantaneous. That is, the code is progressively optimized during execution which means that subsequent builds can be faster purely due to this optimization process. Experiments with HotSpot have shown that it takes somewhere between 5 and 10 builds for optimization to stabilize. The difference in perceived build time between the first build and the 10th for a Daemon can be quite dramatic.

The Daemon also allows more effective in memory caching across builds. For example, the classes needed by the build (e.g. plugins, build scripts) can be held in memory between builds. Similarly, Gradle can maintain in-memory caches of build data such as the hashes of task inputs and outputs, used for incremental building.