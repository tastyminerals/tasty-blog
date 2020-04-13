---
layout: post
title: "Using D Libraries in Files and Projects"
author: tastyminerals
categories: dlang
---

It took me some time to find all the relevant bits and pieces on how to setup a D project.
Ironically, there is plenty of information right there on the official website and D wiki.
But it is not **streamlined**, but rather hidden in the layers of documentation and in some cases scattered all over the place.
As a result a newcomer quickly gets lost.
This post describes how to use external dependencies in your D scripts and projects.

### Installing Prerequisites

You probably already know how to run D scripts from command line.
If not, check out [Running D code like a script](https://wiki.dlang.org/Getting_Started)!

Scripts are nice but when it comes to external dependencies you'll need a build system to make your life easier.
[Many build systems](https://wiki.dlang.org/Build_Tools) support D but the official is DUB.
DUB is the official package manager for D language akin to RubyGems, LuaRocks or pip.
It doesn't come with the language and you need to install it separately with your OS package manager.

* Linux : `sudo pacman install dub`
* macOS: `brew install dub`
* Windows: get a dedicated installer [here](https://github.com/dlang/dub/releases)

That's it!

### Configuring and Running DUB Projects

#### Adding Dependencies to Single-file Scripts

DUB works with single files and with projects of different kinds.
If you need to add some specific library as a dependency to a single file here is how to do it.

Let's randomly pick a dependency from the [D package collection](https://code.dlang.org).

Aaand our pick is **D:YAML** library!
Now, add it as a dependency to the `my_script.d` file.

```d
/+ dub.sdl:
    name "yaml_test"
    dependency "dyaml" version="~>0.8.0"
+/

import std.stdio;
import dyaml;

void main() {
    Node root = Loader.fromFile("test.yaml").load;
    foreach(string word; root["cheatcode"]) {
        write(word ~ " ");
    }
}
```

Add a small `test.yaml` file for input:

```yaml
cheatcode : [show me the money]
```

DUB provides you with several options on how to build and run the file.
The most simple thing you can do is type `dub my_script.d`.

```
> dub my_script.d
show me the money
```

The build artifacts are not cached and each time you call `dub my_script.d`, you'll have to wait.
In order to build the executable from `my_script.d` type `dub build --single my_script.d`.

```
> dub build --single my_script.d
Performing "debug" build using dmd for x86.
tinyendian 0.2.0: building configuration "library"...
dyaml 0.8.0: building configuration "library"...
yaml_test ~master: building configuration "application"...
Linking...
```

Now, you will have a `yaml_test` (or if on Windows `yaml_test.exe`) which you can run.
Did you notice why the executable file is not `my_script`?

In order to run the file right after the build is complete type `dub run --single my_script.d`.

```
> dub run --single my_script.d
Performing "debug" build using dmd for x86.
tinyendian 0.2.0: target for configuration "library" is up to date.
dyaml 0.8.0: target for configuration "library" is up to date.
yaml_test ~master: building configuration "application"...
Linking...
To force a rebuild of up-to-date targets, run again with --force.
Running .\yaml_test.exe
show me the money
```

You can use a different compiler (provided you have it installed) too `dub run --single my_script.d --compiler=ldc2`.

```
> dub run --single my_script.d --compiler=ldc2
Performing "debug" build using ldc2 for x86_64.
tinyendian 0.2.0: building configuration "library"...
dyaml 0.8.0: building configuration "library"...
yaml_test ~master: building configuration "application"...
Running .\yaml_test.exe
show me the money
```

In case your script requires several dependencies just add another line with `dependency` name and `version` number.

#### Adding Dependencies to Projects

When you need more than a single file to solve your problem, it will be easier to organize everything into a project.
DUB does not force you to have a specific directory structure, instead it provides you with good-enough defaults.
It is up to you to override them if necessary.
We shall let DUB do the job for us.
Open up a terminal and type `dub init my_project`.
This will start a short interactive QA session that helps you to configure the project.

```
> dub init my_project
Package recipe format (sdl/json) [json]: sdl
Name [my_project]:
Description [A minimal D application.]: A small D project example.
Author name [tasty]:
License [proprietary]: MIT
Copyright string [Copyright © 2020, tasty]:
Add dependency (leave empty to skip) []: mir-algorithm
Added dependency mir-algorithm ~>3.7.28
Add dependency (leave empty to skip) []: mir-blas
Added dependency mir-blas ~>1.1.13
Add dependency (leave empty to skip) []:
Successfully created an empty project in 'C:\Devel\my_project'.
Package successfully created in my_project
```

Let's explore the project directory structure.

```
my_project/
    source/
        app.d <-- a template file with main function, can be renamed

    dub.sdl <-- dub configuration file
    .gitignore <-- D specific .gitignore file
```

If you open the `dub.sdl` file you'll see the following.

```sdl
name "my_project"
description "A small D project example."
authors "tasty"
copyright "Copyright © 2020, tasty"
license "MIT"
dependency "mir-algorithm" version="~>3.7.28"
dependency "mir-blas" version="~>1.1.13"
```

Now, let's improve the configuration a little by adding compiler flags, library specific configuration and build types.

```sdl
name "runme"
description "A small D project."
authors "tasty"
copyright "Copyright © 2020, tasty"
license "MIT"
dependency "mir-algorithm" version="~>3.7.28"
dependency "mir-blas" version="~>1.1.13"

targetType "executable" // generate executable binary
dflags-ldc "-mcpu=native" // ldc2 compiler flags to improve CPU performance
subConfiguration "mir-blas" "library" // mir-blas specific configuration

// release build type, does several performance optimizations
buildType "release" {
    buildOptions "releaseMode" "inline" "optimize"
    dflags "-boundscheck=off"
}

// executes a unittest block in each project file
buildType "tests" {
    buildOptions "unittests"
}

// debug with optimize
buildType "debug" {
    buildOptions "debugMode" "debugInfo" "optimize"
}

// debug with profile
buildType "debug-profile" {
    buildOptions "debugMode" "debugInfo" "profile"
}
```

In case you're wondering what's that weird sdl format.
That's [simple declarative language](https://sdlang.org) that can be used to write configuration files.
DUB also supports JSON format if you are more comfortable with it.
However, keep in mind that sdl supports comments and less verbose.

Here is JSON version of the above config.

```json
{
    "name": "runme",
    "authors": [
        "tasty"
    ],
    "description": "A small D project.",
    "copyright": "Copyright © 2019, tasty",
    "license": "MIT",

    "targetType": "executable",
    "dependencies": {
        "mir-algorithm": "~>3.7.18",
        "mir-blas": "~>1.1.9"
    },
    "dflags-ldc": ["-mcpu=native"],
    "subConfigurations": {
        "mir-blas": "library",
        "comment": "sub configuration comment"},
    "buildTypes": {
        "release": {
            "buildOptions": ["releaseMode", "inline", "optimize"],
            "dflags": ["-boundscheck=off"]
        },
        "debug": {
            "buildOptions": ["debugMode", "debugInfo", "optimize"]
        },
        "debug-profile": {
            "buildOptions": ["debugMode", "debugInfo", "profile"]
        },
        "tests": {
            "buildOptions": ["unittests"]
        }

    }
}
```

Build the project with `dub build --compiler ldc2 --build release`.

Build & run the project with `dub run --compiler ldc2 --build release`.

Test the project with `dub run --compiler ldc2 --build tests`.

Now, you should be well equipped to work on your shiny new project.

#### Further Reading

* [Getting started with DUB](https://dub.pm/getting_started)
* [DUB command line options](https://dub.pm/commandline.html)