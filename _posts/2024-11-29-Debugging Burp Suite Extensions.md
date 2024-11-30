--- 
title: How to Debug Burp Suite Extensions with IntelliJ
date: 2024-11-29 19:00:00 -0700 # 7:00 PM
author: anthony   
categories: [Development, Burp Suite Extensions]
tags: [dev work, open-source, how-to]    # TAG names should always be lowercase
media_subpath: /assets/posts/debugging-burp-suite-extensions
image:
  path: attachDebugger_dark.png
  alt: Attach Debugger Icon
description: This article guides you through configuring IntelliJ to start a debugging session with Burp Suite seamlessly at the click of a button. Streamline your workflow and enhance your ability to test Burp Suite extensions faster and more efficiently.
---

Burp Suite extension development is a rewarding challenge, but it can be time-consuming without the right tools. It requires a fair amount of experience with Java, it's difficult to write tests without having to mock all Montoya objects, and debugging the code while running Burp Suite can be tedious. After this how-to, you'll have one less thing to think about.

##  Requirements
The following are the requirements for developing and debugging Burp Suite extensions with Intellij. *While familiarity with either the [Montoya Api](https://portswigger.github.io/burp-extensions-montoya-api/javadoc/burp/api/montoya/MontoyaApi.html) or [Wiener Api](https://github.com/PortSwigger/burp-extender-api) is recommended, it's not required for this how-to.*

- [IntelliJ](https://www.jetbrains.com/idea/download/)
- [Java 21 and lower](https://portswigger.net/burp/documentation/desktop/extensions/creating)
- Burp Suite [Community](https://portswigger.net/burp/communitydownload)/[Professional](https://portswigger.net/burp/pro) (*ScanChecks are not allowed within Community Edition*)
- A Burp Suite extensions project written in Java; it does not matter if Montoya Api or Wiener Api are used.

> For this how-to, I'll be using my personal [Boiler Plate project](https://github.com/ahanel13/Burp-Extension-Boilerplate) which was created to help testers start writing extensions without requiring all the setup.
{: .prompt-warning}

## 1. Setting Up Your Project in IntelliJ
1. Make sure you have the project on the machine. If you're following the how-to go ahead and `git clone` or download the repo from [GitHub](https://github.com/ahanel13/Burp-Extension-Boilerplate).
2. Open IntelliJ.
3. Open a new or existing project depending on your situation.
4. Allow IntelliJ to process the project as either gradle or maven

## 2. Building the Jar File for an Extension
Both Maven and Gradle are used when writing Java to help manage project dependencies and automate the build process. For more information: [Difference between Gradle and Maven](https://www.geeksforgeeks.org/difference-between-gradle-and-maven/)

### Maven
Since IntelliJ was told that the project was Maven, it should be fairly simple.
1. Navigate to the `POM.xml` file.
2. `Right-click -> Maven -> download sources` This will download the required dependencies to build the project.
3. Open the Maven lifecycle window in IntelliJ.
4. Double-click `package` to build the jar which can be found in `target/boilerplate-1.0.0.jar`. The target directory is created during the building process.

### Gradle
To build the jar using Gradle, follow these steps:
1. Navigate to the `build.gradle` file.
2. Open the Gradle tool window in IntelliJ.
3. Ensure that all dependencies are downloaded by refreshing the Gradle project.
4. In the Gradle tool window, navigate to `Tasks` > `build`.
5. Double-click on `build` to execute the build task. This will compile the project and create the jar file.
6. The jar file can be found in `build/libs/boilerplate-1.0.0.jar`. The `build` directory is created during the building process.

## 3. Configuring the JDWP Debugging Agent
We'll be using Java JDWP for remote debugging. For some background, a Java Debug Wire Protocol (JDWP) agent allows remote debugging of Java applications by providing the agent's profile to Java when executing a jar.

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

## 4. Setting Up IntelliJ to Debug Burp Suite
We'll be using IntelliJ's ability to execute other tools. To do this:
1. Click on `File -> Settings -> Tools -> External Tools`
2. Add a tool with the `+` button
3. I've named my tool "BurpSuite"
4. Fill out the following
   - `Programs`: this will be the path to your Java executable, i.e. java.exe 
   - `Arguments`: this will be the *JDWP* agent from the previous section prepended with the full path to your Burp Suite Java executable. (`burpsuite.jar`)
5. Make sure the `Open console for tool output` checkbox is checked.
6. Click "Ok" 

![Image of IntelliJ tool configuration](IntelliJToolConfig.png)

## 5. Run the Burp Suite External Tool
To run external tools use the `Tools` menu item in the top left.

1. Run the tool `Tools -> External Tools -> BurpSuite`
  ![Running external tools with IntelliJ](IntelliJExternalTools.png)
1. Attach the debugger within the terminal that pops up. (_**Hint**: it's a button_)
  ![Attach debugger button within run terminal](IntelliJAttachDebuggerBtn.png)
1. Confirm the debugger is attached by looking in the debug windows
  ![Debug windows with target socket](debugWindow.png)

## 6. Actually debugging
1. Add a breakpoint to something that will execute.
  I chose to add a breakpoint to the `initialize(MontoyaApi)` function since that's the first thing that is called by Burp Suite.
  ![MontoyaApi initialize method](initializeMethod.png)

2. Install/Enable the extension jar we built previously.
  ![Add Burp Suite extension menu](addBurpExtensionMenu.png){:weight="50%"}

3. **Verify that the breakpoint was hit!**
  ![Windows within IntelliJ showing debugger context](IntelliJDebugingContext.png)

## 7. Adding More Convenience
> While this configuration is not required, it turns a 4-click action into a 2-3-click action depending on when you last ran the debugger. 
{: .prompt-tip}

1. Create a Debugging Configuration: `Run -> Edit Configurations...`.
2. Add a new configuration with the `+` button. The configuration does not matter, we just need a quicker way (less clicks) to run the tool and IntelliJ does not directly provide one to which we can attach a listener.
3. I chose "Shell Script" and added:
  ```
  echo "Be sure to click attach the listener
  ``` 
  (_the actual config/script does not matter, make it do nothing if you want_)
1. Add a `Before Launch` action with the `+` button and add `External Tools/Burpsuite`.
2. Ensure the `Activate tool window` and `Focus tool Window` boxes are checked.

![Example of Run Configuration](debugBurpRunConfig.png)

## Tips & Tricks
> You can quickly disable and re-enable any extension with `Ctrl + Left Mouse Click` on the extension checkbox.
{: .prompt-tip }

To test any new code changes, you'll have to rebuild the jar and reinstall/re-enable the jar within the Burp Suite instance currently being debugged.

> If you have multiple Burp Suite instances open, only the Burp Suite opened using the debugging profile will have debugging enabled.
{: .prompt-danger }


### Troubleshooting
- **Debugger Fails to Attach**: Ensure that port 5005 is not blocked or already in use by another process. You can use netstat or similar tools to check port availability.

- **JDWP Agent Errors**: Double-check the debugging agent configuration string to ensure there are no typos. Ensure that the Java version matches your Burp Suite requirements.

- **Jar File Not Found**: Ensure you’ve built the jar file correctly and that the file path is accurate. Refresh Maven/Gradle dependencies if needed.

## Conclusion
Debugging Burp Suite extensions with IntelliJ doesn’t have to be difficult. By following this guide, you’ve streamlined your workflow and gained the tools to test your extensions efficiently. Whether you’re a seasoned developer or new to extension development, these steps should help you work smarter, not harder. If you have questions or feedback, feel free to reach out or explore the [Boiler Plate project](https://github.com/ahanel13/Burp-Extension-Boilerplate) for more resources.
