In order to ensure smooth adoption of the RIB architecture, we have built tooling to (1) make RIBs easier to use (2) take advantage by the invariants created by adopting RIBs. Tools we have open sourced are:

- [RIBs Code Generation Plugin for Android Studio and IntelliJ](#ribs-code-generation-plugin-for-android-studio-and-intellij)
- [NullAway: Fast Annotation-Based Null Checking for Java](#nullaway-fast-annotation-based-null-checking-for-java)

# RIBs Code Generation Plugin for Android Studio and IntelliJ

We have created an Android Studio / IntelliJ plugin to generate RIBs scaffolding, making RIB usage and adoption easier. The scaffolded classes have RIB structures in place. Test scaffolding for classes that should have business logic is also generated for use.

After installing the plugin, RIBs can be added with the `New -> New RIB...` command. This generates:
- Scaffolding: [RIBName]Builder, [RIBName]Interactor, [RIBName]Router and [RIBName]View
- Test classes for unit testing: [RIBName]InteractorTest, [RIBName]RouterTest

RIBs can be generated with or without Presenter and View classes.

![The RIB IntelliJ Plugin](https://github.com/uber/ribs/blob/assets/tooling_assets/android-rib-plugin-1.png)

## Plugin Installation Instructions

In Android Studio or Intellij, open `IntelliJ IDEA > Preferences > Plugins` and select `Install Plugins From Disk`. Then install the [RIBs plugin jar](https://raw.githubusercontent.com/uber/RIBs/android-tooling-tutorial/android/tooling/rib-intellij-plugin/rib-tooling-2.png). After this, the plugin will appear in the `New` menu.

![Adding a new RIB from IntelliJ, after having installed the plugin](https://github.com/uber/ribs/blob/assets/tooling_assets/android-rib-plugin-2.png)

## Plugin Build Instructions

To install the plugin locally:
* Run `./gradlew :tooling:rib-intellij-plugin:buildPlugin -Dorg.gradle.configureondemand=true -Dbuild.intellijplugin=true`
* Install the jar file generated within `build`
* Make sure you've installed the correct jar. If you install the wrong jar, you could see runtime crashes.

# NullAway: Fast Annotation-Based Null Checking for Java

NullAway is the tool we built to help eliminate `NullPointerExceptions` (NPEs) in Java code. We use this tool on top of our RIBs stack for static analysis.

The project is open sourced on its own. You can read more about it and download it on the [NullAway Github page](https://github.com/uber/NullAway).