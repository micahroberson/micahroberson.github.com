---
publish: true
layout: post
title: "How To Accept Inbound Email In Any Rails App"
date: 2014-01-26 15:48
comments: true
keywords: rails, ruby on rails, ruby, griddler, thoughtbot, inbound email, email parsing, mandrill
description: A how-to on accepting incoming email with any rack-based application using Thoughtbot's gem, Griddler, in conjunction with a service such as SendGrid or Mandrill.
cover: 
---

Email has been around forever. It's not sexy and it's certainly not new and shiny. BUT, people rely on it like never before and continue to check it even more religiously with the continued evolution of mobile technology. For this reason, Email notifications are a crucial component in many, if not most, web and mobile applications created in the last 5 years. Therefore the importance of being able to accept responses to those [transactional emails](http://blog.mailchimp.com/what-is-transactional-email/) should be very high as well. Consider the following: A user receives a notification email about a coworker commenting on a project via their Gmail app on their mobile phone. Shouldn't they be able to respond to this comment right away without having to open the app?

<!--more-->

### Prereqs

Hopefully you're already using [SendGrid](http://sendgrid.com/), [Postmark](https://postmarkapp.com/) or [Mandrill](http://mandrill.com/) for your transactional emailing needs, but if not, you'll want to sign up with one of them before proceeding. I'm partial to Mandrill, so that's what I'll be using for this tutorial. However, all of these services offer an API for parsing inbound emails, which when triggered, will issue a POST to an endpoint within your app with the contents of the email in a neatly formatted object. 

### Getting Setup

Start off by adding griddler to your Gemfile. [Griddler](https://github.com/thoughtbot/griddler) is a Rails plugin that automatically sets up a route for you, and passes through an object with some high-level parsing options available. 

    gem 'griddler'
    
And install it via:

    $ bundle install

The default route is as follows:

    email_processor POST /email_processor(.:format)   griddler/emails#create


Or, if you'd like to override with a custom path:

    mount_griddler('/api/custom-incoming-email-path')

Griddler expects to find a class called EmailProcessor that defines at least a single class method named <code>process</code>, which accepts a single argument: an object representing the email. You can add this class anywhere, but I'm going to place mine in the lib directory. If this is the first class you're adding to the <code>/lib</code> directory in this project, you'll want to add <code>config.autoload_paths += %W(#{config.root}/lib)</code> to your <code>application.rb</code> file.
In this example, I'm using some code from [Doozie](https://doozie.io) which allows for registered users to respond to comment notifications with a followup comment. 
{% codeblock lang:ruby %}
class EmailProcessor
  def self.process(email)
    # All of your custom logic for handling the email object will be in here!
    
    # First we need to check if the sender is a registered user and if not, just ignore it
    author = User.where( email: email.from ).first
    unless author.nil?
      # Now we need to determine exactly what the email is for. In the case of Doozie, 
      # we accept inbound emails for a number of situations.
      # The destination email address is the key to determining the user's intention.
      if match = email.to.first[:token].match(/^comment-(.*)-(.*)/i)
        commented_model_type, commented_model_id = match.captures
        # Lookup the commented_model to ensure it exists
        commented_model = commented_model_type.titleize.constantize.where( id: commented_model_id ).first
        # Create a new comment model with the body of the email
        comment = commented_model.comments.create(
          message: email.body,
          author: author
          )
      end
    end

  end
end
{% endcodeblock %}

For the example above, we use a destination email address of something like <code>comment-card-52e08ccf38323300020b0000@mail.doozie.io</code>. Where <code>-</code> serves as the delimiter. The first token, <code>comment</code>, is the model to be created, <code>card</code> is the commented_model or the model on which someone just commented, and the last token, <code>52e08ccf38323300020b0000</code> is the ObjectID of the aforementioned <code>card</code>. With this email tokenized, it's easy to lookup the corresponding models and understand that a new comment needs to be created. To setup the routes in the Mandrill , you'll need to start by adding your inbound domain:

{% img width-100 http://cl.ly/image/1v470o2o0J0z/Image%202014-01-28%20at%202.58.50%20PM.png %}

To setup your domain, you'll just need to add a couple MX records to your DNS. Once that's setup and verified, you can start adding Routes:

{% img width-100 http://cl.ly/image/0o3K3w2h2B16/Image%202014-01-28%20at%203.07.41%20PM.png %}

You'll see I setup an inbox-dev route that points to my Pagekite url which is a proxy to a local dev server. This makes testing new features and improvements simple, without compromising production routes. 

After getting your inbound email addresses and routes configured, you're ready to start testing!
    




