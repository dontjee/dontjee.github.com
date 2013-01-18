---
layout: post
title: "Unit Testing with Windows Azure Storage"
description: ""
category: 
tags: []
---
{% include JB/setup %}
When developing against Windows Azure Table Storage, developers will eventually stumble onto the question of how to unit test code written using the Azure SDK. The common answer is to wrap access to table storage in a [Repository Object](http://www.remondo.net/repository-pattern-example-csharp/) or [Query Objects](http://lostechies.com/jimmybogard/2012/10/08/favor-query-objects-over-repositories/).

Then when unit testing, mock those objects to keep your code from actually trying to hit actual table storage. To test the concrete Repository/Query Object, write an integration test against table storage. Some may skip this step because there shouldn't be much logic in these objects.

It works, but 
