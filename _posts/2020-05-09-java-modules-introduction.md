---
layout: post
title: "Java Modules Introduction"
permalink: "java-modules-introduction/"
last_modified_at: 2020-05-09T00:00:00
excerpt: "Package manage classes and modules manage packages"
category: "java"
---

Classes are the basic unit of a program in Java. **Packages** are used to **manage classes** and **modules** are used to **manage packages**. 

Modules were introduced in Java 9. Before that, an application consists of packages that hold classes. It is a problem in large codebases. A public class means that anyone from anywhere could access that class, which can lead to unwanted dependencies. Modules are uniquely named, reusable groups of related packages as well as resources. 

A Module controls which packages should be private for internal use only and which should be public for other modules to use. In modular Java, a public class no longer means that anyone can access that class from anywhere, instead, it is public within a module only, unless module **exports its package** for other modules to use. It will print out a list of Java modules.

From Java 9, the language itself has broken down into modules. Run `java --list-modules` on JDK 9 or above.

```console
$ java --list-modules
java.base@11.0.5
java.compiler@11.0.5
java.datatransfer@11.0.5
java.desktop@11.0.5
java.instrument@11.0.5
.
.
.
```

You can check any module's meta-data with the command: `java --describe-module [module name]`. For example:

```console
$ java --describe-module java.logging
java.logging@11.0.5
exports java.util.logging
requires java.base mandated
provides jdk.internal.logger.DefaultLoggerFinder with sun.util.logging.internal.LoggingProviderImpl
contains sun.net.www.protocol.http.logging
contains sun.util.logging.internal
contains sun.util.logging.resources
```

`exports java.util.logging` the classes in this package are accessible to other modules.<br/>
`requires java.base mandated` this module depends on another module named: mandated.<br/>
`provides ...` this module is providing a service implementation (we won't cover services in this article)<br/>
`contains` this is an internal package, accessible within module only.

## A bare minimum Java module Example

Our sample application will contain two modules, one for utilities called utils and one main application module that will use the utils module. Let's code it up:

Create a directory `JavaModuleExample` and `cd` into it:

```console
$ mkdir JavaModuleExample
$ cd JavaModuleExample/
```

Create root and subdirectories to contain code. **Utils module** will be in `module.utils` directory and **App module** will be in `module.app` directory. Both directories will contain their sub-packages. Let's create them:

```console
$ mkdir -p module.utils/com/module/utils
$ mkdir -p module.app/com/module/app
```

Final Directory Structure would look like:
```
$ tree
.
├── module.app
│   └── com
│       └── module
│           └── app
└── module.utils
    └── com
        └── module
            └── utils

8 directories, 0 files
```
In above structure, `module.app` has package `com.module.app` and `module.utils` has package `com.module.utils`.

### Utils Module
Let's complete `module.utils`. Create a new Java File `Utils.java` in package `com.module.utils` with following `cat` command. It will start waiting for text input.

```
$ cat > module.utils/com/module/utils/Utils.java

```

paste following code in console, press `Ctrl+D` when finished

```java
package com.module.utils;

public class Utils {
    public static int sum (int arg0, int arg1) {
        return arg0 + arg1;
    }
}

[Press Ctrl+D]
```
To tell Java we are writing a module, we need to provide `module-info.java` at the root level of the module. Create the following file again with `cat` command:

```
$ cat > module.utils/module-info.java

```
paste following code in console, press `Ctrl+D` when finished


```java
module module.utils {
    exports com.module.utils;
}

[Press Ctrl+D]
```
In the above file, we are specifying our module, the name of the module is `module.utils`, you can name it whatever you want but is a convention to name module the same way as we name packages.

The Other thing is we are exporting a package `com.module.utils`. We want this package to be available in other modules. If you don't explicitly export it, then by default, it will be only accessible within the module itself.

That's it, we have created a simple module.

### App Module

The Next step is to complete `module.app`. Again, we need `module-info.java` at the root of our App module and since our app module has a dependency on the utils module, we need to specify that dependency in `module-info.java`. Create `module-info.java` with the following command: 

```
$ cat > module.app/module-info.java
```

paste following code in console, press `Ctrl+D` when finished

```java
module module.app {
    requires module.utils;
}

[Press Ctrl+D]
```

In the above file, we need to explicitly tell which modules do we need as a dependency. `requires module.utils` is specifying we need utils module in our app module.

Now Create `Main` class to use `Utils`.

```
$ cat > module.app/com/module/app/Main.java

```

paste following code in console, press `Ctrl+D` when finished

```java
package com.module.app;
 
import com.module.utils.Utils;
 
public class Main {
    
    public static void main(String[] args) {
        System.out.println(Utils.sum(1,2));
    }
}

[Press Ctrl+D]
```

we are calling `sum(...)` function from `Utils` class which exists in the util module.

Finally, our application would look like: 
```
$ tree
.
├── module.app
│   ├── com
│   │   └── module
│   │       └── app
│   │           └── Main.java
│   └── module-info.java
└── module.utils
    ├── com
    │   └── module
    │       └── utils
    │           └── Utils.java
    └── module-info.java

8 directories, 4 files
```

### Compile

Run following command to compile code: 

```
$ javac -d outDir --module-source-path . $(find . -name "*.java")
```
`javac` is java compiler that will compile java code to byte code <br/>
`-d outDir` is where our compiled code will go <br/>
`--module-source-path .` we are specifying module source path, which is current directory `.` <br/>
`$(find . -name "*.java")` we are specifying which files to compile (all files ending with .java)

### Run

Run following command to run code: 

```
$ java --module-path outDir -m module.app/com.module.app.Main
3
```
`java --module-path outDir` will specify compiled modules directory <br/>
`-m module.app/com.module.app.Main` specifying which class to run, `Main` class in `module.app` 

`3` is printed as a result, which is the result of `Utils.sum(1,2)`, the code we have written in our Main class in the App module. 

That was a basic introduction to modules in Java. As a side note, I would like to mention that you should not confuse Java modules with maven modules or IntelliJ IDEA modules. Maven and IntelliJ IDEA have their concepts of modules too. Also, IntelliJ IDEA supports Java modules too see their blog post [here](https://blog.jetbrains.com/idea/2017/03/support-for-java-9-modules-in-intellij-idea-2017-1/){:target="_blank"}.
