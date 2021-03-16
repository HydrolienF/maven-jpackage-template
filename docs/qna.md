# Q&A

### Q: Can you give me a few more details about how this works?

A: Maven plugins are used to copy all of the project dependencies into a folder, generate a slimmed-down JVM, and
then generate a platform-specific installer. The [pom.xml](https://github.com/wiverson/maven-jpackage-template/blob/main/pom.xml)
is heavily commented!

### Q: Any Tips for Windows?

A: First, make sure you **set a custom Windows installer UUID for your project**, as described in 
the [pom.xml](https://github.com/wiverson/maven-jpackage-template/blob/main/pom.xml)!

This UUID is used to uniquely identify the app as YOUR app by the Windows installer system, and is critical for 
allowing users to seamlessly upgrade. You can quickly [grab a UUID of your own](https://www.uuidgenerator.net/) and pop that value in instead. By default jpackage will generate a UUID automatically, but this automatic UUID is easily
regenerated with minor changes to your application, breaking the Windows installer upgrade chain.

The [Windows GitHub workflow](https://github.com/wiverson/maven-jpackage-template/blob/main/.github/workflows/maven-build-installer-windows.yml)
for this project downloads the Wix Installer toolkit and adds it to the path to automatically build the Windows
installer. On your local dev machine just install [WiX Toolset](https://wixtoolset.org/)
locally instead - it'll be a lot faster.

### Q: I'm getting errors when I'm trying to use javafx.web?

A: GitHub won't allow cloning a template if the source has files over 10mb in size. The javafx.web components basically
bundle a full native web browser under the covers. As of JavaFX 15 the javafx.web.jmod is roughly 25mb in size. If you
need it, you can [download it](https://gluonhq.com/products/javafx/) and install it in the JavaFX projects in your local
project.

If you are delivering a project that essentially amounts to a Java web application bundled as a web application, instead of
bundling JavaFX and a WebKit browser, I would suggest creating a small preferences UI using Swing and 
the [System Tray API](https://docs.oracle.com/javase/tutorial/uiswing/misc/systemtray.html). With a nice modern look and feel
such as [FlatLaf](https://www.formdev.com/flatlaf/), you can create a very nice preferences panel and then use the
standard Java [Desktop API](https://docs.oracle.com/javase/9/docs/api/java/awt/Desktop.html) to just launch the user's
browser to the local URL.

If you drop me a note that this is something you are interested in, I may go ahead and create a template for this... :)

### Q: Tell me a bit about the Linux version?

A: The current GitHub Workflow for the Linux build runs on a GitHub Ubuntu instance, and by default it generates a
amd64.deb file. jpackage supports other distribution formats, including rpm, so if you want a different packaging format
you can tweak the
[GitHub Action for the Ubuntu build](https://github.com/wiverson/maven-jpackage-template/blob/main/.github/workflows/maven-build-installer-unix.yml)
and
the [jpackage command for Unix](https://github.com/wiverson/maven-jpackage-template/blob/main/src/packaging/unix-jpackage.txt)
to generate whatever you need. As long as you can find the right combination of configuration flags for
[jpackage](https://docs.oracle.com/en/java/javase/15/docs/specs/man/jpackage.html) and can set up the GitHub Actions
runner to match, you can generate whatever packaging you might need. If you need, you could set up several combinations
of Maven profile and GitHub Action to generate as many different builds as you wish to support. For example, you could
support generating macOS .dmg and .pkg files, Windows .msi and .exe, Linux .deb and .rpm in several different binary
formats.

### Q: What about macOS Signing?

A: You will likely need to add additional options to ship properly on macOS - most notably, you will want to sign and
notarize your app for macOS to make everything work without end user warnings. Check out tools such
as [Gon](https://github.com/nordcloud/gon)
or [this command-line signing tutorial](https://blog.dgunia.de/2020/02/12/signed-macos-programs-with-java-14/).

### Q: Can I generate macOS installers on Windows, or Windows installers on macOS? Or macOS/Windows on Linux?

A: [No](https://openjdk.java.net/jeps/392), but this project uses GitHub workflows to generate
[macOS](https://github.com/wiverson/maven-jpackage-template/blob/main/.github/workflows/maven-build-installer.yml)
and
[Windows](https://github.com/wiverson/maven-jpackage-template/blob/main/.github/workflows/maven-build-installer-windows.yml) (and Linux)
installers automatically, regardless of your development platform. This means that (for example)
you could do your dev work on Linux and rely on the GitHub Actions to generate macOS and Windows builds. If you need
help, reach out to [ChangeNode.com](https://changenode.com/).

You still should (of course) do platform specific testing on your apps, but that's a different topic.

### Q: Does this support auto-updating, crash reporting, or analytics?

A: No... for that, you should check out [ChangeNode.com](https://changenode.com/)!

### Q: I'd rather use shell scripts.

A: Ok. Check out [JPackageScriptFX](https://github.com/dlemmermann/JPackageScriptFX) - the original shell scripts
used as a reference when I initially started work on this project.

### Q: I didn't realize JavaFX was so cool - any pointers?

Sure - here's
my [personal list of cool JavaFX resources](https://gist.github.com/wiverson/6c7f49819016cece906f0e8cea195ea2), which
includes links to a few other big lists of resources.

### Q: Any Maven tips?

If you are not familiar with the standard Maven build lifecycle, you are highly encouraged to review the documentation
["Introduction to the Build Lifecycle"](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
to understand how Maven builds work.

The project also uses os-activated, platform-specific
[profiles](https://maven.apache.org/guides/introduction/introduction-to-profiles.html) for configuration - really cool
for setting up platform-specific stuff.

### Q: Wait, didn't this project use to try to completely modularize the application first?

A: Correct.

There were two previous versions of this template, and both strategies were discarded as impractical in the real world.

The first approach was to **try to create a shaded jar** (a single jar containing all of the project dependencies) and then
use jdeps to create a module-info.java and add it to the shaded jar. This failed to account for things like multi-release
jars and jars with service declarations. While it worked for trivial applications, it fell apart with more complex real
world usage. In particular, attempting to integrate Spring Boot into the application caused many issues to appear.

The second attempt took a more sophisticated approach - **using jdeps to automatically modularize all of the project dependencies.** 
A new [`collect-modules` goal](https://github.com/wiverson/jtoolprovider-plugin/blob/main/collect-modules-doc.md) 
was added. In brief, the plugin would walk through the entire Maven dependency tree and sort all of the declared dependencies
into folders. The dependencies that already were modularized (both basic and multi-release jars) went into one folder,
and ordinary non-modularized jars went into another. The plugin would then attempt to use jdeps to automatically generate
module-info.java and add the compiled module-info.classes into each jar.

Unfortunately, this also failed. There were numerous errors, such a circular references between libraries (e.g. slf4j and
logback). Some jars would only work when jdeps generated open module-info.java files, and others would only work when
jdeps generated module-info.java files with package level exports. Many, many jars would have large numbers of packages
exposed, which mean that the resulting module-info.java files contained many, many entries. The error messages and the
resolution for these messages were very, very confusing. The terminology around modules is unfortunately very inaccessible
to a typical Java developer - for example, a "static" declaration in a Java module-info.java is used to denote a concept
similar to the Maven notion of "provided" - but unfortunately jdeps fails to run if a needed module is declared as
a transitive reference. Even if it's optional.

In the end, even with the plugin, the error messages and the resolution of those messages is just simply not something
a typical Java developer can be expected to understand or fix.

Which brings us back to this project. The end result is effective the same - just change the `<jvm.modules>javafx.media,javafx.controls,javafx.fxml,java.logging</jvm.modules>` declaration in the pom.xml to
specify the JVM modules you need and just skip all trying to modularize your app.

For now, I would consider the creation and use of module-info.java to effective be a system programming interface
for the JDK itself and system-level modules such as JavaFX itself, where the entire dependency tree is very carefully
enforced - and likely includes native code. For ordinary developers, just enjoy the benefits of a trimmed down custom
JVM and don't worry about it. After all the challenges working with
 [(incorrectly written!) module-info.java files](https://github.com/sormuras/modules/tree/main/doc/suspicious)
I found just in the Spring Boot dependency graph, I would highly suggest that maintainers for ordinary, non-native
Java libraries remove their existing module-info.java files.

It's a pity, as I think that if jdeps, jlink, and jpackage were set up to me more user-friendly, I think it would
be a very interesting system that might lead to slimmed down, cloud-friendly applications. For me, the
acid test for Java module adoption is probably Spring Boot. It's very, very popular. From a technical standpoint, it
uses technologies such as reflection that are often notoriously tricky when working with static compilation heavily.
An end user should be able to simply start working with Spring Boot and at at most a few commands to their build
process. Clearly projects such as [Spring Native](https://github.com/spring-projects-experimental/spring-native)
shows there is an interest in this space.
 