---
published: false
layout: post
title: "Bootstrapping a Single Page Backbone App with Rails"
date: 2013-07-08 09:56
updated: 2013-07-08 09:56
comments: true
keywords: rails, backbone, backbone.js, require.js, bootstrapping, bootstrap data, spa, single page app
description: How to reliably load bootstrap data in your single-page application from a Rails application.
---

Rails and Backbone.js have a long history together. In fact, Jeremy Ashkenas designed Backbone.js to work with a Rails app from the get-go. However, there are a few intricacies involved in actually getting a Rails app up and running with Backbone.js if you intend to use Require.js as well(which I would highly recommend - see <a href='/when-not-to-use-requirejs-or-another-amd-framework'>When Not to Use Require.js</a>).
<!--more-->