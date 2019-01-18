---
layout: "post"
title: " 404s and web.config errors using git in WSL"
description: ''
category: null
tags: []
---

I've run into two problems with local development recently and I wanted to share the resolution to the problems in case it helps others.

#### IIS Express 404s
The first problem was after a fresh clone using git in a bash prompt under Windows Subsystem Linux (WSL). When I started the full framework, asp.net application, the response was always a 404. IIS Express could not see content or application code in that directory no matter what I changed. The fix was to either re-clone using git in a regular windows command prompt, or copy all the files to another folder manually created in windows explorer. Something about the permissions of the parent folder don't work well with IIS Express when created through the WSL git executable.

#### Web.config not recognized
The second problem was with the web.config file not being recognized. I again had cloned the repository using git in a bash prompt under WSL. The file existed on disk but IIS Express was not reading it and applying the configuration. The fix for this problem was to rename the file in WSL bash to match the casing in the .csproj file.

For example, on disk the file was cloned as `Web.config` but the .csproj entry was: 
```
    <Content Include="web.config">
      <SubType>Designer</SubType>
    </Content>
```

Note the mismatch in the first character casing. Once I renamed the file to be all lowercased as the .csproj file expected, everything worked. I believe this issue has something to do with case sensitivity differences between WSL and Windows. 