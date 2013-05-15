---
layout: post
title: "Why I Use Test Driven Development"
description: ""
category: 
tags: []
---
{% include JB/setup %}
[Test Driven Development](http://en.wikipedia.org/wiki/Test-driven_development) (TDD) is one of the more polarizing techniques in software engineering. People love it or they hate it, often without ever actually trying it. I feel like it's a useful tool in my programming toolbox, helpful for a multitude of reasons, but it is not the silver bullet of software development. Some of the reasons I practice Test Driven Development are:

##### TDD guides decisions and helps to develop sufficiently architected code.
TDD encourages "good enough" solutions. When you're focused on doing just enough to make the tests pass, you aren't going to create complex, over architected solutions. Instead you'll end up with a clean design that you know works for at least one client, the tests.

A common complaint about TDD is actually the opposite of my point, which is that it results in poorly designed code. It's easy to fall into the trap of writing a failing test, making it pass, then continuing on, ignoring the Refactor step. Left alone, this technical debt can fester and create a mass of poorly designed code. However, if the developer is diligent in refactoring I believe the design will come out as good or better than one designed completely up front. You develop an architecture that makes sense to the client (the tests) and fits the context.

The biggest problem for me when doing Test Driven Development is the Refactor step. I am often tempted to skip or make weak attempts at the refactoring. In order to truly reap the benefits of TDD, you must be diligent in this step.

#####TDD Encourages decoupled code
The process of Test Driven Development encourages the developer to defer work to interfaces and then mock outputs of those interface functions to test a given method in isolation. Doing so results in decoupled code that is flexible enough to accommodate future change. However, care must be taken to craft those interfaces in a logical way that will make sense to future developers. Often times it is hard to name these pieces and personally this is where I end up spending a fair bit of time. Others will spend more time reading my code than I spent writing it so I want to be careful and name things well. Additionally, with refactoring tools like Resharper the cost of changing the name several times as I get a feel for what the object is doing is minimal.

#####Allows testing in isolation.
Writing tests allows you to test classes in isolation, which allows for a quicker testing loop. Rather than firing up the entire application in a debugger and stepping through the code, you can just write a test that exercises the code to prove it works. This is especially useful for logic heavy classes where common errors like bounds checking or parsing problems can be a pain to test manually. With TDD, less time spent in the debugger will often translate to quicker development.

#####TDD gives some confidence that new changes have not broken system
Finally, test first development can provide protection against many (not all) regression bugs. Because most of your code has test coverage, many regression problems will be found and resolved early in the development cycle, usually before they are even committed. The quicker those bugs are found, the easier and cheaper they are to fix. However, it is not a perfect system and will not prevent all regression bugs, nor should it be relied on to do so. TDD should be coupled with full system tests to verify things actually work from a user's perspective in order to have sufficient confidence the code does what it should.

#####TDD forces you to understand what you're doing before doing it
Writing the tests first requires you to know how the code works and how to add your new feature to existing code. This can feel like a burden at first but in practice it ends up being a good thing. Without tests, the temptation is to hack in the new feature and ship it. With tests you can't really do that, TDD requires you to know what test you want to write next.

A common practice to help figure out where you're going is "spiking". Ignoring tests, you implement as much of the feature as you need to get a feel for what needs to be written and where the code should go. Then you throw that work away and rewrite it using TDD. This second pass goes much quicker than the first and you have a good idea of what tests to write and where you are headed. Both give you a better understanding of the code, leading to more robust code with less regression bugs.
