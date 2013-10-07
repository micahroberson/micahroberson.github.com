---
publish: true
layout: post
title: "Absolute Links with Backbone.js and Rails"
date: 2013-07-18 09:15
comments: true
keywords: rails, backbone.js, backbone, require.js, single page app, refresh, push-state, history
description: Methods for allowing users to refresh within a Backbone.js app or load a page within a Backbone.js app from an an absolute url
---

Ideally user's won't ever need to refresh your single page app. With the use of websockets, or simulated websockets via polling, this is avoidable nearly 100% of the time. However, there are some cases where it would be very handy for the user to be able to refresh the page and return to exactly where they were. Perhaps the most important scenario is actually for developers! Making a code change and having to click through several links to actually test the code leaves something to be desired.
<!--more-->
To start we need to make sure all Backbone URL/non-API requests get routed to load up the app appropriately. From Rails' perspective, its rendering the same view, home#index, for every non-API request but making sure the relative path gets passed along with it. To keep it simple, I've just added a catch-all route, but it may be better to whitelist specific routes that match up with your Backbone.js routes. Notice I'm using Devise and also include some "api" scoped routes to illustrate how this particular app is setup.

    devise_for :users
    scope :path => "api" do
      resources :projects
      ...
      resources :posts
    end
    constraints :format => "html" do
      match '*path', :controller => 'home', :action => 'index'
    end

Within the Home Index view, we just need to ensure we pass down that params[:path] value to the backbone router. This is a snippet from index.html.erb and also where I would include any bootstrapped data that the app would always need before starting up.

    <script>
      require([
        'jquery',
        'underscore',
        'backbone',
        'router'
      ], function($, _, Backbone, Router){
        $(function() {
          // Set the current user
          currentUserModel.set(#{Rabl::Renderer.new('users/current_user', current_user, :view_path => 'app/views').render});
          // Navigate to the current route. One could also pull this from window.location.pathname if no fancy Rails manipulation is req'd
          Router.navigate('<%= params[:path].html_safe %>', {trigger: true});
        });
      });
    </script>

Assuming all the backbone routes are implemented and setup correctly, this is all that needs to be done! Depending on the type of view, it may need some additional logic to work properly from both an absolute url and also from a relative link within the app. The difference being that it may be dependent on resources being previously loaded which won't be available should the user arrive at the view from an absolute path. There's a couple of options for this scenario: a. Use a Railsy strategy in the router and run a before_filter on every route to load necessary dependencies before even getting to the view code, b. Pass the model to the view if its available and don't if its not, but equip the view to react accordingly. For the former, there's a handy Backbone.js mixin called [backbone.routefilter](https://github.com/boazsender/backbone.routefilter), which will call router.before and/or router.after around each route.

    before: () ->
      route = Backbone.history.fragment

      if route.indexOf('dashboard') != -1 || route.indexOf('posts') != -1
        if undefined == currentUserModel || currentUserModel.isNew()
          @listenToOnce currentUserModel, 'change', () => Backbone.history.loadUrl(route)
          return false

This before filter simply checks for the currentUserModel to be set and returns false if it isn't while setting up a callback to re-run the same route once the model IS loaded.  The former solution mentioned above might look like this:

    render: () ->
      if !@model
        @model = new userModel _id: @user_id
        @listenToOnce @model, 'sync', @renderProfile
        @model.fetch null, { success: () => @listenTo @model, 'change', @renderProfile }
        usersCollection.add @model
      else
        @listenTo @model, 'change', @renderProfile
        @renderProfile()

      @$el.html _.template(profileTemplate)
      @

Depending on the app and if there are some common dependencies throughout, the before filter method might work well, while the latter method might work better for other scenarios. Personally I've used the latter for views that are normally accessed via a list view, ie. a user profile page or detail view, but should still be accessible via the absolute url.
