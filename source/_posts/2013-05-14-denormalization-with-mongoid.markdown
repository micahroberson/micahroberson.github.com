---
published: false
layout: post
title: "Denormalization with Mongoid"
date: 2013-05-14 08:51
comments: true
keywords: rails, mongoid, mongodb, embedded_in, embeds_many, denormalizing
description: Correctly configure embedded_in/embeds_many relations with Mongoid and Mongoid::Alize.
---

[Mongoid::Alize](https://github.com/dzello/mongoid_alize) is a great little gem that provides a simple, light DSL for denormalizing attributes across all types of Mongoid relations. Under the hood, it implements all of the necessary callbacks on both sides of the relation and provides the selected fields in an easy to use hash.
<!--more-->

    include Mongoid::Alize
    ...
    belongs_to :user
    alize :user, :name, :profile_thumbnail

One of the latest features I've added for 