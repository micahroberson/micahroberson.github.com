---
publish: true
layout: post
title: "Preventing Duplicate Content | Rails SEO Tip"
date: 2013-08-17 11:13
comments: true
keywords: seo, duplicate content, ruby on rails, rails, canonicalization, multiple urls, preferred domain
description: Canonicalization is the SEO concept of redirecting multiple URLs to a single master URL. Learn how to implement a solution in Ruby on Rails to prevent duplicate content complications to improve your SEO.
---

It's useful to allow users to reach your site by visiting both the naked domain, greeksquare.com, as well as the www. prefixed version, www.greeksquare.com. However, this puts search engines in a sticky situation because from their perspective, these are two different sites with the same exact content. Fortunately, there's a simple solution for site's built with any Rack based web server(ie. Ruby on Rails, Sinatra, etc.): [rack-rewrite](https://github.com/jtrupiano/rack-rewrite). Rack-rewrite is a simple middleware for applying rewrite rules (redirects) and we're going to use it to redirect a couple of prefixed domains to the naked domain. By pointing all variations of the same content to one place, we eliminate the competition between the different URLs, and also boost the overall relevancy and popularity.

For this example, I'm going to use some snippets from the main Rails app behind [GreekSquare](https://greeksquare.com). Let's begin by adding the rack-rewrite gem:

    gem 'rack-rewrite'

Once the gem is installed, its as simple as adding some rules to production.rb. If you plan on having several rules, it would be best to put this in an initializer.

    config.middleware.insert_before(Rack::Lock, Rack::Rewrite) do
      r301 %r{.*}, 'http://greeksquare.com$&', :if => Proc.new {|rack_env|
        rack_env['SERVER_NAME'] == 'www.greeksquare.com' || rack_env['SERVER_NAME'].include?('herokuapp')
      }
    end

In this example I'm rewriting all 'www.greeksquare.com' and 'greeksquare.herokuapp.com' requests to simply be the naked domain 'greeksquare.com'.  Also notice the redirect status code being used, r301, which indicates the site has 'Moved Permanently'. More info on status codes [here](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html). 

It's also a good idea to make sure you set the Preferred domain in Google's Webmaster Tools. It's easy to get setup with Webmaster Tools, and once you do just visit the Site Settings page where you'll find the form below:

{% img center /images/preferred_url.png %}
