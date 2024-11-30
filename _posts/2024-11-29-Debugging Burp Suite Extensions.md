--- 
title: How to debug Burp Suite Extensions with Intellij
date: 2024-11-29 19:00:00 -0700 # 7:00 PM
author: anthony   
categories: [Development, Burp Suite Extensions]
tags: [dev work, open-source, how-to]    # TAG names should always be lowercase
media_subpath: /assets/posts/debugging-burp-suite-extensions
image:
  path: attachDebugger_dark.png
  alt: Attach Debugger Icon
description: This article guides you through configuring IntelliJ to seamlessly start a debugging session with Burp Suite at the click of a button. Streamline your workflow and enhance your ability to test code faster and more efficiently.
---

Developing Burp Suite extensions can take time. It requires a fair amount of experience with Java, it's difficult to write tests without having to mock all Montoya objects, and debugging the code while running Burp Suite can be tedious. After this walkthrough, hopefully you have one less thing to think about.

##  Requirements
The following are requrements to develop *and* debug Burp Suite extensions.

- [Intellij](https://www.jetbrains.com/idea/download/)
- [Java 21 and lower](https://portswigger.net/burp/documentation/desktop/extensions/creating)
- Burp Suite [Community](https://portswigger.net/burp/communitydownload)/[Professional](https://portswigger.net/burp/pro) (*ScanChecks are not allowed within Community Edition*)
- A Burp Suite extensions project written in Java, does matter if Montoya Api or Wiener Api are used.

> For this walkthough, I'll be using my personal [Boiler Plate project](https://github.com/ahanel13/Burp-Extension-Boilerplate) which was created to help testers start writing extensions without requiring all the setup.
{: .prompt-warning}

## Open Project with Intellij
1. Make sure you have the project on the machine. If you're following the walkthrough go ahead and `git clone` or download the repo from [GitHub](https://github.com/ahanel13/Burp-Extension-Boilerplate).
2. Open Intellij..
3. Open new project. 
4. Allow Intellij to process the project as either gradle or maven

## Build Jar
Both Maven and Gradle are used when writing Java to help manage project dependencies and automate the build process. For more information see [Difference between Gradle and Maven](https://www.geeksforgeeks.org/difference-between-gradle-and-maven/)

### Maven
Since Intellij was told that the project was Maven, it should be fairly simple.
1. Navigate to the `POM.xml` file.
2. `Right-click -> Maven -> download sources` This downloads required dependencies to build the project.
3. Open the Maven lifecycle window in Intellij.
4. Double-click `package` to build the jar which can be found in `target/boilerplate-1.0.0.jar`. The target directory is created during the building process.

### Gradle
To build the jar using Gradle, follow these steps:
1. Navigate to the `build.gradle` file.
2. Open the Gradle tool window in Intellij.
3. Ensure that all dependencies are downloaded by refreshing the Gradle project.
4. In the Gradle tool window, navigate to `Tasks` > `build`.
5. Double-click on `build` to execute the build task. This will compile the project and create the jar file.
6. The jar file can be found in `build/libs/boilerplate-1.0.0.jar`. The `build` directory is created during the building process.

## How to use Java Debugging Agents
For some background, we're going to use a Java Debug Wire Protocol (JDWP) agent, which allows remote debugging of Java applications. This is done by providing the agent's profile to Java when executing a jar.

*Debug Agent Configuration:*
```
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
```

| Configuration         | Description                                   |
| --------------------- | --------------------------------------------- |
| `transport=dt_socket` | Use socket transport for communication.       |
| `server=y`            | Enable the JVM to listen for a debugger.      |
| `suspend=n`           | Don’t suspend the JVM; continue running.      |
| `address=*:5005`      | Listen on port 5005 for debugger connections. |

## Confiuring to Intellij Execute Burp with JDWP Agent
We'll be using Intellij's abaility to execute other tools. To do this:
1. Click on `File -> Settings -> Tools -> External Tools`
2. Add a tool with the `+` button
3. I've named my tool "BurpSuite"
4. Fill out the following
   - `Programs`: this will be the path to your java executable, i.e. java.exe 
   - `Arguments`: this will the *jdwp* agent from previous section prepended with the full path to your Burp Suite java executable.
5. Make sure "Open console for tool output" is checked.
6. Click "Ok" 

![Image of Intellij tool configuration](intellijToolConfig.png)

## Run the Burp Suite External Tool
The way external tools are intended to be run is by using the `Tools` menu item in the top left.

1. Run the tool `Tools -> External Tools -> BurpSuite`
  ![Running external tools with Intellij](intellijExternalTools.png)
2. Attach the debugger within the terminal that pops up. (_**Hint**: it's a button_)
  ![Attach debugger button within run terminal](intellijAttachDebuggerBtn.png)
3. Confirm the debugger is attached by looking in the debug windows
  ![Debug windows with target socket](debugWindow.png)

### Adding More Convience
> While this configuration is not required, it turns a 4 click action into a 2-3 click action depending on when you last ran the debugger. 
{: .prompt-tip}

1. Create a Debugging Configuration: `Run -> Edit Configurations...`
2. Add a new configuration with `+` button. The configuration will not matter, as we just need simple way to quickly run the tool and Intellij does not provide one.
3. I chose "Shell Script" and added:
  ```
  echo "Be sure to click attach listener
  ``` 
  (_the actual config/script does not matter, make it do nothing if you want_)
4. Add a "Before Launch" action with the `+` button and add `External Tools/Burpsuite`
5. Ensure "Activate tool window" and "Focus tool Window` boxes are checked.

![Example of Run Configuration](debugBurpRunConfig.png)

## Actually debugging
1. Add a breakpoint to something that will execute.
  I chose to add a breakpoint to the `initialize(MontoyaApi)` function since that's the first thing that is called by Burp Suite.
  ![MontoyaApi initialize method](initializeMethod.png)

2. Install/Enable the extension jar we built previously.
  ![Add Burp Suite extension menu](addBurpExtensionMenu.png){:weight="50%"}

3. **Verify that the breakpoint was hit!**
  ![Windows within IntelliJ showing debugger context](intellijDebugingContext.png)

## Notes
> You can quickly disable and re-enable any extension with `Ctrl + Left Mouse Click` on the extension checkbox.
  {: .prompt-tip }

In order to test new code, you'll have to rebuild the jar and reinstall/re-enable the jar within the Burp Suite that is being debugged.

> If you have multiple Burp Suite instances open, only the Burp Suite opened using the debugging profile will have debugging enabled.
{: .prompt-dranger }
