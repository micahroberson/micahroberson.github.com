---
layout: post
published: true
title: "Foundation Nav w/ jPanelMenu"
comments: true
date: 2013-05-04 10:21
categories: 
keywords: rails, single page app, foundation, zurb, jpanelmenu, mobile navigation, flyout menu, facebook style navigation
description: Producing a more mobile friendly navbar in Foundation with a little help from JPanelMenu.
---

[Foundation](http://foundation.zurb.com/) provides some excellent out-of-the-box support for responsive layouts and navigation. However, in building out the new mobile interface for [GreekSquare](http://www.greeksquare.com/), we weren't satisfied with the default style. 
<!--more-->

{% img width-50 right //s3.amazonaws.com/micahroberson/greeksquare-mobile-expanded.png %}

This isn't horrible but it also isn't great. I strongly believe a great ui is extremely important and provides for a ton of value to the end-user - especially in the market we're going after. I set out to find a suitable javascript implementation of a mobile navigation similar to the Facebook mobile app. I tried several different jquery plugins, but they were all extremely invasive and resulted in messy code with clunky javascript animations that didn't fare too well when I tested on an actual device. Finally I came across [jPanelMenu](http://jpanelmenu.com/). jPanelMenu is a lightweight(10kb minified) jquery plugin that lets you pass it any navigation element and flips it around to display in a Facebook style sidenav menu.

{% img width-50 left //s3.amazonaws.com/micahroberson/greeksquare-mobile-new-expanded.png %}

Much, much better than before - both aesthetically, and from a usability standpoint. The end product came out with very few html changes to existing navbar. I ran into a small hiccup when using the fixed implementation of Foundation's navbar, but that was solved by add the following css to shrink it at the same rate as the jPanelMenu expands (ease-in-out at 300ms in my case). 



    [data-menu-position="open"] .container .fixed {
      left: 200px;
    }

    .container header .fixed {
      @include transition(ease-in-out left .3s);
    }

