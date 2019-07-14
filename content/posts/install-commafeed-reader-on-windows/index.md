---
title: "Install CommaFeed Reader on Windows"
date: 2018-02-10T19:51:04-07:00
tags: ["CommaFeed", "Windows", "RSS"]
---

Although [Google Reader](https://www.google.com/reader/about/) ceased operations in 2013, I still rely on [RSS](https://en.wikipedia.org/wiki/RSS) every day and won't check a website regularly unless it offers a feed.

<!--more-->

Many [open-source self-hosted web-based feed readers](https://en.wikipedia.org/wiki/Comparison_of_feed_aggregators) exist, but [CommaFeed](https://www.commafeed.com/#/welcome) has some unique advantages over the others. Although it relies on a web browser to present its interface, the program does not require the installation of a separate web server or database, making it very easy to install on Windows.

1. Download the [Java Runtime Environment (JRE)](https://java.com/en/download/) and install it with default options

1. Create a new folder to hold CommaFeed, such as **C:\Programs\CommaFeed**

1. Download the [latest release of commafeed.jar](https://github.com/Athou/commafeed/releases) and save it as **C:\Programs\CommaFeed\commafeed.jar**

1. Open the [CommaFeed sample configuration file](https://raw.githubusercontent.com/Athou/commafeed/master/config.yml.example) in a web browser and
	* Right-click anywhere on the web page and select **Save as...**
	* Set the **Save as type** to **All Files**
	* Save the file as **C:\Programs\CommaFeed\config.yml** (remove any extraneous **.example** and **.txt** extensions)
{{< figure src="save-config-yml.png" alt="Save configuration file">}}
	
1. Open **C:\Programs\CommaFeed** in Windows Explorer
{{< figure src="commafeed-folder.png" alt="Commafeed folder in Windows explorer">}}

1. Right-click **C:\Programs\CommaFeed\commafeed.jar** and select **Create shortcut** to create a shortcut

1. Right-click the shortcut **C:\Programs\CommaFeed\commafeed.jar - Shortcut** and select **Properties**, then set the (*very long*) **Target** and **Run** values and press the **OK** button
	* **Target:**
    java -Djava.net.preferIPv4Stack=true -jar "C:\Programs\CommaFeed\commafeed.jar" server "C:\Programs\CommaFeed\config.yml"
	* **Run:**
    Minimized
	* Press the **OK** button
	{{< figure src="commafeed-shortcut-properties.png" alt="CommaFeed shortcut properties">}}

1. Double-click the shortcut **C:\Programs\CommaFeed\commafeed.jar - Shortcut** to run the server and press the **Allow access** button if a firewall warning appears
{{< figure src="windows-security-alert.png" alt="Windows firewall security alert">}}

1. Open http://localhost:8082 and login with user: **admin** and password: **admin**

1. To run CommaFeed automatically when Windows starts
	* Open **C:\Programs\CommaFeed** in Windows Explorer
	* Open the **Start** menu and type **shell:startup** to open the startup folder
{{< figure src="shell-startup.png" alt="Open the startup folder">}}
	* Drag and drop the shortcut **C:\Programs\CommaFeed\commafeed.jar - Shortcut** into the startup folder
    * Commafeed will now run automatically when Windows starts
