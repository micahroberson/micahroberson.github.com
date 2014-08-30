---
publish: true
layout: post
title: "Unit Testing Backbone Views with Mocha in a RequireJS App"
date: 2014-08-29 16:14
comments: true
keywords: backbone.js, require.js, yeoman, mocha, javascript, coffeescript, testing, unit-testing
description: A breakdown of a working Mocha, RequireJS, Backbone, Yeoman configuration for unit-testing.
cover: 
---

Require, Mocha, Backbone and Yeoman seem to be popular enough when you look at them individually. But get them in the same room as Grunt, LiveReload and CoffeeScript and you might have some issues. I couldn't find an example out there for this specific setup so I thought I should share what I've pieced together.

<!--more-->

### The App Setup
The app was originally started via the [Yeoman Backbone.js Generator](https://github.com/yeoman/generator-backbone), but the Gruntfile has had so many revisions since that it hardly resembles what was spit out on day one. I try to be pragmatic about writing tests, so they usually don't come entirely before the app itself, but rather during or potentially after depending on the situation. However, because Grunt is so flexible, it's pretty easy to get this workflow modified to fit whatever your style is. To kick things off I'm just going throw out some configs so we all know what we're dealing with.

A chunk from Bower.json so you have an idea of what makes up the app:
{% codeblock lang:json %}
  "jquery": "~1.11.0",
  "underscore": ">=1.5",
  "backbone": "~1.1",
  "requirejs": "~2.1.5",
  "requirejs-text": "~2.0.5",
  "layoutmanager": "0.9.5",
  "pikaday": "1.2.0",
  "fastclick": "1.0.1"
{% endcodeblock %}

Here's some of the relevant pieces from my Gruntfile. For the sake of brevity, I've shown only test-related tasks.
{% codeblock lang:coffeescript %}
  coffee:
    options:
      bare: true
      force: true
      spawn: false
    modified:
      files: [
        expand: true
        cwd: '<%= yeoman.app %>/scripts'
        src: '**/*.coffee'
        dest: '.tmp/scripts'
        ext: '.js'
        filter: isModified
      ,
        expand: true
        cwd: 'test/spec'
        src: '**/*.coffee'
        dest: '.tmp/spec'
        ext: '.js'
        filter: isModified
      ]

  connect:
    options:
      port: SERVER_PORT
      hostname: '0.0.0.0'
    livereload:
      options:
        middleware: (connect) ->
          return [
            modRewrite(['!(\\..+)$ /index.html [L]'])
            lrSnippet
            mountFolder(connect, '.tmp')
            mountFolder(connect, yeomanConfig.app)
          ]
    test:
      options:
        port: 9001
        middleware: function (connect) {
          return [
            mountFolder(connect, '.tmp')
            mountFolder(connect, 'test')
            mountFolder(connect, yeomanConfig.app)
          ]

  mocha_phantomjs:
    all:
      options:
        run: false
        urls: ['http://localhost:<%= connect.test.options.port %>/index.html']
        spawn: false

  watch:
    options:
      nospawn: true
      livereload: true
    coffee:
      options:
        spawn: false
      files: ['test/spec/**/*.coffee']
      tasks: ['coffeelint:modified', 'coffee:modified', 'mocha_phantomjs']

  grunt.registerTask 'serve', (target) ->
    if target == 'test'
      return grunt.task.run [
        'clean:server'
        'coffee'
        'haml'
        'symlink'
        'connect:test'
        'mocha_phantomjs'
        'watch'
      ]
{% endcodeblock %}

Something to note: changes in .coffee files shouldn't force *all* .coffee files to be recompiled but instead only the one that changes. This goes for tests as well - we don't need to rerun the whole suite when a single test changes(one scenario I'm yet to get figured out is when a non-test .coffee changes, the respective spec should run). But that's essentially the setup. I won't walk through each of those blocks, but if you have any questions about anything, please do ask!

### Da' Spec Runner!
So this guy pulls in the Require.js app and kicks off the whole dealio. Any `require.config` options provided here override whatever you've got set in the app config file. Another plus to this setup, is the single require config file for *both* the test environment and the main app. Sometimes you do want to change a few paths for [DI](http://en.wikipedia.org/wiki/Dependency_injection) purposes, but we've got a simple way to do that here.

{% codeblock lang:html %}
  <head>
    <base href="/"/>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
    <title>Mocha Spec Runner</title>
    <link rel="stylesheet" href="lib/mocha/mocha.css">
    <script src="lib/mocha/mocha.js"></script>
    <script src="lib/chai.js"></script>
    <script src="lib/sinon.js"></script>
  </head>
  <body>
    <div id="mocha"></div>
    <script>mocha.setup({ui: 'bdd', bail: false, globals: ['Modernizr']})</script>
    <script>var expect = chai.expect</script>
    <script src="scripts/require-config.js"></script>
    <script data-main="spec/runner" src="../bower_components/requirejs/require.js"></script>
  </body>
{% endcodeblock %}

{% codeblock lang:coffeescript %}
  require.config
    baseUrl: 'scripts/'
    urlArgs: 'now=' + Date.now()
    map:
      '*':
        'collections/posts' : '../spec/mocks/collections/posts'

  specs = [
    '../spec/models/post.js'
    '../spec/views/posts/list'
  ]

  require specs, () ->
    if window.mochaPhantomJS
      mochaPhantomJS.run()
    else
      mocha.run()
{% endcodeblock %}

So one beef I have with this setup is that the `specs` array needs to be updated with any new specs if you want them to run. I'll get around to this eventually, and post an update when I do. Or if anybody has any thoughts please send 'em my way!

There's also a check for mochaPhantomJS. grunt-mocha-phantomjs let's us run in the browser as well if you need to debug any pesky situations. 

{% codeblock lang:coffeescript %}
  define (require) ->
    require '../../../spec/helper'
    Layoutmanager      = require 'layoutmanager' # Backbone Layout Manager
    PostsListView      = require 'views/posts/list'
    ListView           = require 'views/common/list'
    PostsCollection    = require 'collections/posts'
    CommentMock        = require '../../mocks/models/comment'

    view               = null

    # We need to call startup Backbone Layout Manager to get views rendering
    # as they do in the wild.
    Backbone.Layout.configure
      manage: true

    # Stub fetch on PostsCollection
    sinon.stub PostsCollection, 'fetch', (options = {}) ->
      data = [
        {
          id: 1
          published: false
          initiator_id: 1
          initiated_at: '08/1/2014'
          body: 'Sample body'
        }
      ]
      PostsCollection.reset data, { silent: true }
      options.success() if options.success?
      PostsCollection.trigger 'sync', PostsCollection, { data: data }
      $.Deferred().resolve().promise()

    describe 'PostsListView', ->
      before () ->
        view = new PostsListView()

      it 'extends ListView', ->
        expect(PostsListView.__super__).to.deep.equal(ListView.prototype)

      describe '.initialize', ->
        it 'sets @collection', ->
          expect(view.collection).to.deep.equal(PostsCollection)

        it 'calls @listenTo on @collection for the change event', ->
          expect(view.collection._events.change[0].context).to.deep.equal(view)

      describe '.afterRender', ->
        it 'calls @fetchModels', ->
          fetchModelsSpy = sinon.spy(view, 'fetchModels')
          view.render()
          expect(fetchModelsSpy.calledOnce).to.equal(true)

        it 'appends the row html to .table-body', ->
          view.render()
          expect(view.$el.find('.table-row').length).to.equal(1)

      describe 'events', ->
        describe 'when date start changes', ->
          it 'calls onChangeDateRange', ->
            spy = sinon.spy view, 'onChangeDateRange'
            view.render()
            view.$('.date-start').val('2014-10-10').change()
            expect(spy.calledOnce).to.equal(true)
            spy.restore()
        describe 'when date end changes', ->
          it 'calls onChangeDateRange', ->
            spy = sinon.spy view, 'onChangeDateRange'
            view.render()
            view.$('.date-end').val('2014-10-10').change()
            expect(spy.calledOnce).to.equal(true)
            spy.restore()
        describe 'when .search input changes', ->
          it 'calls onKeyupSearchInput', ->
            spy = sinon.spy view, 'onKeyupSearchInput'
            view.render()
            view.$('.search input').val('RAP').keyup()
            expect(spy.calledOnce).to.equal(true)
            spy.restore()

{% endcodeblock %}

### So Let's Break It Down
The first thing you'll notice is the dependencies to get the view to render essentially on it's own, no parent app attached. Luckily require takes care of all of the child dependencies, but for the pieces higher up the stack, we need to bring them in here. For this particular app, we only need Backbone Layout Manager, the view we're testing(PostsListView), the view PostsListView extends(ListView), and the PostsCollection(a singleton). I threw in a model mock for the CommentModel just to show how that can be done in a reusable manner.

Next up you'll notice the stub on the `PostsCollection.fetch` method. Feel free to take this guy out, and you can run with live server data. Generally it's better to stub xhr requests for two reasons: it's much faster, and these are unit tests so we need to minimize any outside code being pulled in. The stub itself is pretty generic though, and easy to modify if you have any additional logic. For example, I've got a collection with a fetchPage method that keeps state and provides a wrapper around `fetch`. In that case it still made sense to stub the fetch function(rather than the wrapper), but to add some additional logic for handling the requested page parameters.

So for each `describe` block we want a clean state. Hence the `view = new PostsListView()` in the `before` block. We go on to check some setup logic, event bindings and reactions, and then to the actual render method. Much of the meat of the render method in this case is actually handled by the view from which PostsListView inherits: ListView. For that reason we test much of that code in the ListView spec. Along the same lines, this becomes a sort-of integration test, pulling in the collection, ListView and it's own code. But only for a few lines! We want to keep this as unit-testy as possible, and leave the integration testing for later(the next post!). 

### Summing It Up
After all is said and done, we've got a setup that let's you work on specs and have them run automatically. Huge productivity points right thur.
I know this post was a little all over the place, so please do ask for any clarification. I'll see if I can get a gist together with all of the parts together for easier viewing!




### 
