---
layout: post
title: "Choose Your HTML Selectors Wisely"
description: ""
category: 
tags: []
---
{% include JB/setup %}
##### The basics
When getting started with web development the sheer amount of selector options can be overwhelming. The choice to use the tag name, id, class, child selector, etc. is confusing. Without a well defined plan the css for an app can quickly become unwieldy and modifications become nightmarish. In this post I'm going to lay out how I like to work and why.

##### Simple CSS Architecture
Rule #1 - No IDs. People [have](http://oli.jp/2011/ids/) [written](http://screwlewse.com/2010/07/dont-use-id-selectors-in-css/) [before](https://github.com/stubbornella/csslint/wiki/Disallow-IDs-in-selectors) about the pitfalls of using ids in css. Even the CSSLint tool discourages their use. IDs are the most specific way to define styles for an element that can be defined in a CSS file. This makes overloading tricky because of CSS specificity rules. Once you've defined an ID with CSS rules you cannot override it via a class later. Your only options are to redefine the ID later with the updates or to try to be more specific by defining the parent element as part of the selector. In either case you've started down the road to painful CSS maintenance headaches.

There is a better way! As much as possible, stick to using classes only for defining layout and style. The classes are reusable and can be extended later making for a flexible design.

Rule #2 - Do not overload the meaning of a class
l- for layout classes

Rule #3 - Separate js and test classes
