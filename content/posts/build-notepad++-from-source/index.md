---
title: "Build Notepad++ From Source"
date: 2018-04-11T23:15:05-07:00
tags: ["C++", "Notepad++", "Windows"]
---

[comment]: # (	https://github.com/notepad-plus-plus/notepad-plus-plus										)
[comment]: # (	https://github.com/notepad-plus-plus/notepad-plus-plus/issues/4474							)
[comment]: # (	https://github.com/notepad-plus-plus/notepad-plus-plus/pull/4583							)
[comment]: # (	https://github.com/notepad-plus-plus/notepad-plus-plus/issues/1228#issuecomment-388978543	)
[comment]: # (	https://github.com/bfridkis/Building-Notepad-Plus-Plus-from-Source-A-Beginners-Guide		)

I don't try to build [Notepad++](https://notepad-plus-plus.org/) from source very often, and I've never had much luck following
the instructions on the [github page](https://github.com/notepad-plus-plus/notepad-plus-plus), so here's how I do it instead.

<!--more-->

* Most of the [instruction text](https://github.com/notepad-plus-plus/notepad-plus-plus) explains how to build **SciLexer.dll** with and without boost, but I've never actually needed to do that.
Instead, just put a recent copy of **SciLexer.dll** into the correct folder and build Notepad++ by itself.

* Strictly speaking, this is not the right way to do it, but it has worked well enough for me that I have never noticed any missing functionality or other issues.

* Although the [official build instructions](https://github.com/notepad-plus-plus/notepad-plus-plus) refer to **VS2013**,
I was unable to build the solution using Visual Studio Community Edition **2013**, which should be similar.
However, it built perfectly the first time without any trouble using Visual Studio Community Edition **2015**, so that's what I recommend here.

* Notepad++ comes in both portable and installer versions, but I prefer to use the installer because it automatically adds an **Edit with Notepad++** option to the right-click file context menu.
However, either version will work. After building **notepad++.exe** from source, just overwrite whichever version has been installed with the new executable.

* Many useful plugins, such as [Python Script](http://npppythonscript.sourceforge.net/), are not available in x64 versions, so it makes sense to build the x86 version instead.

Here are the steps to build from source and replace an existing installation:

1. Download the newest [Notepad++ Installer 32-bit x86](https://notepad-plus-plus.org/download/) file and install it to the default location **C:\Program Files (x86)\Notepad++**

1. Download or clone the [Notepad++ source code](https://github.com/notepad-plus-plus/notepad-plus-plus) and extract it to **C:\work\npp**

1. Download and install [Microsoft Visual Studio Community Edition 2015](https://go.microsoft.com/fwlink/?LinkId=532606).

1. Run Visual Studio and open the solution file **C:\work\npp\PowerEditor\visual.net\notepadPlus.vcxproj**.

1. Set the build target to create either a **debug** build or a **release** build:
	* select **Unicode Debug** and **x86** to create a **debug** build

	* select **Unicode Release** and **x86** to create a **release** build

1. Press **F7** to build the solution, which will create **notepad++.exe** inside one of these build output folders:
	* **C:\work\npp\PowerEditor\visual.net\Unicode Debug** (for a **debug** build)

	* **C:\work\npp\PowerEditor\bin** (for a **release** build)

1. Copy the file **SciLexer.dll** from **C:\Program Files (x86)\Notepad++** into the same build output folder as **notepad++.exe** in step 6.

1. Close any currently running instances of Notepad++.
	
1. Press **F5** in Visual Studio to run the project and make sure Notepad++ starts and runs.

1. Copy the newly created version of **notepad++.exe** from one of the build output folders into **C:\Program Files (x86)\Notepad++** to overwrite the previously installed version.

1. Run **C:\Program Files (x86)\Notepad++\notepad++.exe** to make sure everything works as expected.
