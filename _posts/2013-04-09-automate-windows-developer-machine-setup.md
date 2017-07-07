---
layout: post
title: "Automate Windows Developer Machine Setup"
description: ""
category: 
tags: []
---
{% include JB/setup %}
With any serious software project you'll quickly realize developer machine setup is non trivial. What I've experienced on projects in the past is someone decides to document the process (Yay!) but the documentation is quickly forgotten and is out of date by the time someone goes to use it (Boo!). Even in the best case where the documentation is up to date the process of downloading and installing required software manually is tedious, boring and error prone. Finishing your machine setup in a day or two was considered an accomplishment. Another popular option is maintaining a developer machine system image. We attempted this as well but if developers had different hardware it fell apart. Additionally the system image tended to get out of date just like the documentation.

##### Enter Chocolatey
As painful as those options were, they were pretty much the only options until [Chocolatey](http://chocolatey.org/) came along. Chocolatey is a package manager for windows, much like apt-get for Linux or homebrew for OSX. Chocolatey is built on powershell and provides a command line interface to install programs. For example, installing Node JS is as simple as running `cinst nodejs.install`. Additionally, Chocolatey can automate enabling windows features, ruby gems or python packages. To enable ASP.Net 4.5 on Windows 8, run `chocolatey WindowsFeatures IIS-ASPNET45`.

Most installations are as simple as running a one line `cinst` command. However, if you wish to customize an installation you can, assuming the installer supports it. For example, the [Visual Studio 2012 package](http://chocolatey.org/packages/VisualStudio2012Ultimate) by default does not include the full installation to avoid a system restart on some machines. However, this is configurable via the `/AdminFile` install argument. This optional argument should contain the path to an `admindeployment.xml` file. The specification for this file is not very well documented by Microsoft, but if you just wish to install the full Visual Studio 2012, [this gist will do just that](https://gist.github.com/dontjee/3981697). The full command to install Visual Studio 2012 in its entirety is

{% highlight bat %}
IF NOT EXIST "C:\Program Files (x86)\Microsoft Visual Studio 11.0\Common7\IDE\devenv.exe" (
       @powershell -NoProfile -ExecutionPolicy unrestricted -Command "((new-object net.webclient).DownloadFile('https://raw.github.com/gist/3981697/46dd422c495e7d5e0cb97adcb0912834246e2c60/AdminDeployment.xml', 'C:\AdminDeployment.xml'))"
       call cinst VisualStudio2012Ultimate -installArgs '/Passive /NoRestart /AdminFile C:\AdminDeployment.xml /Log C:\vs.log' -overrideArguments -force
)
{% endhighlight %}

The commands above check to see if Visual Studio is installed. If not, the AdminDeployment.xml file is downloaded to the root of the C:\ drive and then Visual Studio is installed using its settings instead of the Chocolatey defaults.

##### Putting it all together
To get Chocolatey installed inside a batch script, run
{% highlight bat %}
powershell -NoProfile -ExecutionPolicy unrestricted -Command "iex ((new-object net.webclient).DownloadString('http://bit.ly/psChocInstall'))"

SETX /M PATH "%PATH%;%systemdrive%\chocolatey\bin"

SET PATH=%PATH%;%systemdrive%\chocolatey\bin
{% endhighlight %}

This will ensure that Chocolatey is installed and add it to your PATH permanently (`SETX`) and for the current session (`SET`). You can then use any of the chocolatey commands like `cinst` to setup the environment.

##### Other Notes
One thing I do stress is to make your setup script repeatable so that it can be run multiple times without harmful effects on the machine. For example if I needed to make sure git was installed I would run something like

{% highlight bat %}
IF NOT EXIST "%systemdrive%\Program Files (x86)\Git\cmd" (
  call cinst git.install
  SET PATH=%PATH%;%systemdrive%\Program Files (x86)\Git\cmd
  SETX /M PATH "%PATH%;%systemdrive%\Program Files (x86)\Git\cmd"
  call cinst git-credential-winstore
)
{% endhighlight %}

This will only install git if it is not already installed, but Chocolatey would do that for us as well. The important thing this snippet does is not re-add git to the PATH, which is unnecessary and potentially harmful.

One other thing I would recommend is using the setup script to share out environment changes to the rest of the team. When a new requirement is added to the project, add it to the setup script and let everyone know they need to rerun the script. This will help ensure the script is always up to date, and can be run any number of times without affecting the environment.

##### Wrapping Up

With the addition of Chocolatey we have the power to script a full machine setup on Windows. Projects that I work on now have a predictable `machinesetup.bat` script that a developer can run on a bare-bones machine to get up and running automatically in an hour or so. This is a huge improvement over what was previously a multi-day, error prone process.
