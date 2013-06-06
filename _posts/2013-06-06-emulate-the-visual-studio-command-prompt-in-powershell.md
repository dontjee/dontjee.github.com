---
layout: post
title: "Emulate The Visual Studio Command Prompt In PowerShell"
description: ""
category: 
tags: []
---
{% include JB/setup %}
The Visual Studio Command Prompt provided when you install Visual Studio adds lots of useful commands to the PATH of the current prompt. You don't have to remember where `msbuild.exe` or `mstest.exe` are located just call the commands. However, Visual Studio only ships with a regular command prompt. There is no corresponding PowerShell command prompt.

All hope is not lost for PowerShell fans though. The Visual Studio Command Prompt works by calling the `vcvarsall.bat` file that ships with Visual Studio to update the `PATH` for the current session. We can take advantage of that same script when PowerShell starts to update the `PATH` for the PowerShell session by modifying the default PowerShell profile.

First we need to open the PowerShell profile file for editing. This file is located at `~\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1` If you already have PowerShell open you can call `start $PROFILE` from the prompt. This command will open the profile for you in your default editor.

Next paste these lines at the end of the file.
{% highlight powershell %}
# Move to the directory where vcvarsall.bat is stored
pushd 'C:\Program Files (x86)\Microsoft Visual Studio 11.0\VC'

# Call the .bat file to set the variables in a temporary cmd session and use 'set' to read out all session variables and pipe them into a foreach to iterate over each variable
cmd /c "vcvarsall.bat&set" | foreach {
  # if the line is a session variable
  if( $_ -match "=" )
  {
    $pair = $_.split("=");

    # Set the environment variable for the current PowerShell session
    Set-Item -Force -Path "ENV:\$($pair[0])" -Value "\$($pair[1])"
  }
}

# Move back to wherever the prompt was previously
popd
{% endhighlight %}

Save the file, and reopen a new PowerShell prompt. You should have a fully functioning Visual Studio Powershell Prompt.

Finally, if you don't want to load the Visual Studio variables for every prompt, move the lines above to a file in your path and execute that file when you want the variables `. thefilethathasthescriptabove.ps1`.
