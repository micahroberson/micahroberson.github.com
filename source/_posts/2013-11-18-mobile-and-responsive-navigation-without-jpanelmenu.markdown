---
publish: true
layout: post
title: "Mobile and Responsive Navigation without jPanelMenu"
date: 2013-11-18 08:36
comments: true
keywords: project management, doozie, doozie.io, product development
description: How to build a fast and functional slide-out, panel-style navigation menu using minimal javascript and hardware accelerated CSS for great mobile performance.
---

In my previous post, [Mobile Navigation using Foundation and jPanelMenu](http://localhost:4000/foundation-nav-w-jpanelmenu), I explained how to transform a standard [Foundation](http://foundation.zurb.com) navbar into a more mobile friendly, Facebook-style flyout panel. We initially built this for [GreekSquare](https://greeksquare.com) and the original implementation was fast and smooth. However, as the complexity of our app grew and as we began to test the mobile version on more devices, we quickly found out the performance was less than stellar. Animations were jagged and laggy, and the DOM was polluted with duplicate elements created by the [jPanelMenu](http://jpanelmenu.com/) plugin. Although the jPanelMenu animations are written in a hardware-accelerating fashion, the menu was still too slow. I decided to remove the plugin entirely and build the simple mechanism manually. 

<!--more-->

This is the entirety of the SCSS responsible for the new menu:
{% codeblock lang:sass %}
  .slidemenu-body {
    overflow-x: hidden;
    position: relative;
  }
  .slidemenu {
    @include transition(all 0.3s ease);
  }
  .slidemenu-left {
    left: -200px;

    -webkit-backface-visibility: hidden;
    -moz-backface-visibility: hidden;
    -ms-backface-visibility: hidden;
    backface-visibility: hidden;

    -webkit-perspective: 1000;
    -moz-perspective: 1000;
    -ms-perspective: 1000;
    perspective: 1000;

    @include transform(translate3d(0,0,0));

    &.slidemenu-open {
      @include transform(translateX(200px));
    }
  }

  // For GreekSquare, we show the whole navbar when the user isn't on a phone/tablet and
  // hide the button to collapse/open the navbar
  @media only screen and (min-width: 769px) {
    .slidemenu-left {
      left: 0px;
    }
    .subnav-btn {
      display: none;
    }
  }
{% endcodeblock %}

Now all we have to do is toggle the slidemenu-open class on both the body and the sidenav elements.

{% codeblock lang:javascript %}
  var $elems = $('body, #subnav');

  // Open the menu when the icon is clicked
  $('.subnav-btn').on('click', function(e) {
    $elems.toggleClass('slidemenu-open');
  });

  // Close the menu when a link within is clicked, and still follow the
  // link (don't stop propagation)
  $(document).on("click", "#subnav.slidemenu-open a", function(e) {
    $elems.removeClass('slidemenu-open');
  });

  // Close the menu and stop propagation on the click event to prevent the action on
  // whatever element the user actually clicked
  $(document).on("click", "body.slidemenu-open #main", function(e) {
    e.stopPropagation();
    e.preventDefault();
    
    $elems.removeClass('slidemenu-open');

    return false;
  });
{% endcodeblock %}

Here's the finished product - mobile version closed & open and the desktop version:

{% img width-50 left /images/gs_nav_closed.png %}
{% img width-50 left /images/gs_nav_open.png %}

{% img width-100 center /images/gs_desktop.png %}
