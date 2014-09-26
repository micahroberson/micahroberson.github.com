---
publish: false
layout: post
title: "Functional Testing with Grunt, Backbone, RequireJS, and Mocha"
date: 2014-09-22 07:25
comments: true
keywords: backbone.js, require.js, yeoman, mocha, javascript, coffeescript, testing, functional testing
description: A continued breakdown of working with Mocha, RequireJS, Backbone, Yeoman configuration for writing functional tests.
cover: 
---

In my last post, I disected the process of getting Mocha into your Backbone/RequireJS project to write unit-tests. In this next post, we'll dive into writing functional tests for the same type of single-page application(SPA). 
<!--more-->

> Unit tests tell a developer that the code is doing things right; functional tests tell a developer that the code is doing the right things.  
> [Unit Testing versus Functional Tests](http://www.softwaretestingtricks.com/2007/01/unit-testing-versus-functional-tests.html)

## Why We Need Functional Tests
Unit tests are crucial for setting expectations of individual components in a system and ensuring they deliver the right output. Functional tests are responsible for ensuring all of the components of a system work as-expected when running together and as a whole from a user's perspective. These tests are extremely important in preventing bugs introduced by refactoring, especially in continuous deployment setups. Functional tests serve as a snapshot of the state of the application, and when they're all passing, you know the app is stable and user's can succesfully do eveything exactly as they could before your changes.

Imagine making a change to a common component, let's call it a table or listing view, that many, many views inherit from throughout the app. Changes to this base view are sure to happen, but you most likely don't how many different derived classes it has, or anything about how they are implemented. Functional tests are the only way to be sure the changes you make are succesfull globally, and not just in the few cases you actually know about and are patient enough to test manually. Unit tests can go a long way when it comes to writing *good* inheritance code, but functional tests can offer a much-needed confidence boost when making extremely impactful changes.

## The Setup
The specs for these functional tests are almost identical to those I used for [unit-testing in my previous post](/unit-testing-backbone-views-with-mocha-in-a-requirejs-app). One key difference, is the introduction of [Squire.js](https://github.com/iammerrick/Squire.js/) for mocking dependencies. Squire should actually help a lot with the unit-testing as well! Dependency Injection and RequireJS within a testing framework is surprisingly tough, but Squire provides an easy way to mock modules on a test-by-test basis.

{% codeblock lang:coffeescript %}
  'use strict'
  define (require) ->
    require 'spec-helper'
    Squire = require 'squire'

    injector = new Squire()
    injector.mock 'collections/posts', require('mocks/collections/posts')

    PostsListView           = require 'views/posts/list'
    UsersCollection         = require 'collections/users'
    PostsCollection         = require 'collections/posts'
    Post                    = require 'models/post'
    backboneLayoutmanager   = require 'backboneLayoutmanager'

    view                    = null
    currentUser             = UsersCollection.getCurrentUser()

    Backbone.Layout.configure
      manage: true

    describe 'updating the contents of a RAP', ->
      before ->
        view = new RapsPriceBreakdownView(id: 1)
        view.render()
{% endcodeblock %}


{% codeblock lang:coffeescript %}
{% endcodeblock %}

{% codeblock lang:coffeescript %}
{% endcodeblock %}
