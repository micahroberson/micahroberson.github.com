---
layout: post
published: false
title: "Foundation Nav w/ jPanelMenu"
comments: true
date: 2013-05-04 10:21
categories: 
keywords: rails, single page app, foundation, zurb, jpanelmenu, mobile navigation, flyout menu, facebook style navigation
description: Producing a more mobile friendly navbar in Foundation with a little help from JPanelMenu.
---

[Foundation](http://foundation.zurb.com/) provides some excellent out-of-the-box support for responsive layouts and navigation. However, in building out the new mobile interface for [GreekSquare](http://www.greeksquare.com/), we weren't satisfied with the default style. 
Here's the desktop navbar:
{% img //s3.amazonaws.com/micahroberson/greeksquare-desktop-navigation.png %}

Default Mobile navbar provided by Foundation with minimal styling (collapsed and expanded):
{% img left //s3.amazonaws.com/micahroberson/greeksquare-mobile-collapsed.png %}
{% img right //s3.amazonaws.com/micahroberson/greeksquare-mobile-expanded.png %}

This isn't 'horrible' but it also isn't great. We believe a great interface provides for a ton of value to the end-user, especially in the market we're going after. So I set out to find a suitable javascript implementation of a mobile navigation similar to the Facebook mobile app. I tried several different jquery plugins, but they were all extremely invasive and resulted in messy code with clunky javascript animations that didn't fare too well when I tested on an actual device. Finally I came across [jPanelMenu](http://jpanelmenu.com/). jPanelMenu is a lightweight(10kb minified) jquery plugin that lets you pass it any navigation element and flips it around to display in a Facebook style sidenav menu.

Here's how the GreekSquare menu turned out:
{% img left //s3.amazonaws.com/micahroberson/greeksquare-mobile-new.png %}
{% img right //s3.amazonaws.com/micahroberson/greeksquare-mobile-new-expanded.png %}

Much, much better than before - both aesthetically, and from a usability standpoint. 