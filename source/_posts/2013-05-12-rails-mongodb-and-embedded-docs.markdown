---
published: true
layout: post
title: "MongoDB Schema Design"
date: 2013-05-12 08:01
updated: 2013-05-12 08:01
comments: true
keywords: rails, mongoid, mongodb, embedded_in, embeds_many, denormalizing
description: Thoughts on MongoDB and schema design.
---

I'm a big fan of MongoDB. I understand that it's not great for every dataset out there, nor would I attempt to make it work with a highly relational dataset just because its shiny and new. What I do find rather appealing about MongoDB is the non-rigid document schema and the different mindset you have to have while developing. The non-rigid schema is important because it allows for very agile development and fast iterations. This doesn't mean I condone adding and removing fields like crazy. I think I still make pretty calculated decisions when it comes to adding new fields to a collection just because I've been in the RDBMS world for so long - which is a good thing! 
<!--more-->
The different mindset I mentioned, came about as a result of a conversation I had at [MongoDB Days SF](http://www.10gen.com/events/mongodb-san-francisco-2013). One of the presenters made a point about modeling the data in the database in the same way as you display it in the app. This is a completely different approach than one would take with a classic RDBMS backed application. With an RDBMS, you're most likely going to attempt to normalize the entire schema and then potentially denormalize certain fields out on a case-by-case basis. We do all of this with the assumption that normalization is ideal, and redundancy is bad. I think this allusion comes from two things: a. storage space actually used to be an issue due to hardware constraints and thusly cost. and b. you can relatively easily access any piece of data with the use of joins and/or a number of other techniques in an RDBMS. With MongoDB, I'll layout the design of a particular view and decide what the main domain model is, and if the view is going to be a list format, there's probably some additional metadata from another model that needs to be displayed alongside(i.e. a list of comments, each showing the author thumnail picture and name). At this point, you need to denormalize that metadata in order to avoid the N+1 beast. Although denormalization is necessary for efficient queries, its not all that difficult to maintain nor is it a downside. I much prefer the overall approach to app development with MongoDB and find it much easier to relate the inner-workings to the UI. 
