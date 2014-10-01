---
publish: true
layout: post
title: "Functional Testing with Grunt, Backbone, RequireJS, and Mocha"
date: 2014-09-22 07:25
comments: true
keywords: backbone.js, require.js, yeoman, mocha, javascript, coffeescript, testing, functional testing
description: A continued breakdown of working with Mocha, RequireJS, Backbone, Yeoman configuration for writing functional tests.
cover: 
---

In my last post, I dove into the process of getting Mocha into your Backbone/RequireJS project to write unit-tests. In this next post, we'll dive into writing functional tests for the same type of components. 
<!--more-->

> Unit tests tell a developer that the code is doing things right; functional tests tell a developer that the code is doing the right things.  
> [Unit Testing versus Functional Tests](http://www.softwaretestingtricks.com/2007/01/unit-testing-versus-functional-tests.html)

## Why We Need Functional Tests
Unit tests are crucial for setting expectations of for the different methods and behavior of individual components on a granular level. Functional tests operate at a higher-level and ensure the component as a whole *functions* the way it should. Oftentimes functional tests do involve testing the composition of multiple components, and end up serving as a poor man's integration test. Which is fine! True integration tests are also important, because they ensure everything, end-to-end, is functioning correctly. Both functional and integration tests are extremely important in preventing bugs introduced by refactoring, or deeply embedded system changes.

It's pretty common to modify a mixin or base class that many other components either include or inherit from. But when you do this, especially when changing the public API, you have to be sure every integration still functions as it should. This can be tricky business, and you usually end up searching through the codebase to see where method 'x' is called and how it's used. While you're still going to have to do this(and should), we can alleviate the stress associated with making scary changes by writing functional tests! These tests serve as a snapshot of the state of the component and it's role in the application as a whole. So when we refactor an existing component, we can re-run the tests and ensure everything still works just as it did before. Unit tests can go a long way when it comes to writing *good* modules and inheritance logic, but functional tests can offer a much-needed confidence boost when making big, impactful changes.

## The Setup
The specs for these functional tests are almost identical to those I used for [unit-testing in my previous post](/unit-testing-backbone-views-with-mocha-in-a-requirejs-app). One key difference, is the use of [Sinon]() for mocking server requests. Request json is stored in separate files(for each request) to keep it organized and maintainable. 

{% codeblock lang:coffeescript %}
  before do
    estimatesJSON = require('text!json/estimates.json')
    server = sinon.fakeServer.create()
    server.autoRespond = true
    server.respondWith('GET', EstimatesCollection.url(),
      [200, {"Content-Type":"application/json"}, estimatesJSON]
    )
  end
{% endcodeblock %}

This is a subset of the tests for this particular view, but you get the idea. Reset the collection and re-render the view in a beforeEach block to ensure we start with a clean slate, and click through to carry out the appropriate actions. The jQuery assertions come from the [Chai jQuery](http://chaijs.com/plugins/chai-jquery) libary. You'll also notice the helper methods at the bottom for easily performing repeated actions. In fact, we probably could have abstracted even more code into similar functions to keep the assertion logic cleaner.

{% codeblock lang:coffeescript %}
  describe 'updating the price breakdown of an estimate', ->
    beforeEach ->
      EstimatesCollection.fetch({async: false})
      view = new EstimatePriceBreakdownView(id: 1)
      view.render()
      enableEditMode()

    describe 'when updating a line item value', ->
      it 'enter value for field labor', ->
        expect(view.model.lineItemsCollection.models).to.have.length(6)
        view.$('.line-item:first').click()
        $('.line-item.modal [name=amount_cents]').val('$1000').change()
        $('.line-item.modal .save').click()

        expect(view.$('.line-item:first [name=amount_cents]').val()).to.equal("$1000")

    describe 'when adding a line item', ->
      it 'adds the line item to the table', ->
        addLineItem()

        expect(view.$('.line-items .input-row')).to.have.length(7)
        expect(view.$('.line-items .input-row:last label')).to.have.text('Test')
        expect(view.$('.line-items .input-row:last [name=amount_cents]')).to.have.value('$300')

      it 'does not add the line item to the table', ->
        view.$('.add-line-item').click()
        $('.line-item.modal .cancel').click()

        expect(view.$('.line-items .input-row')).to.have.length(6)

    describe 'when adding a change order', ->
      it 'adds the change order to the table', ->
        addChangeOrder()

        expect(view.$('.change-orders .change-order')).to.have.length(1)
        expect(view.$('.change-orders .change-order:last [name=number]')).to.have.value('CO-1')
        expect(view.$('.change-orders .change-order:last [name=amount_cents]')).to.have.value('$100')
        expect(view.$('.change-orders .change-order:last [name=received_at]')).to.have.value((new moment()).format('MM/DD/YYYY'))

      it 'does not add the change order to the table', ->
        view.$('.add-change-order').click()
        $('.change-order.modal .cancel').click()

        expect(view.$('.change-orders .change-order')).to.have.length(0)

  enableEditMode = ->
    view.$('.edit').click()

  addChangeOrder = ->
    view.$('.add-change-order').click()
    $('.change-order.modal input[name=number]').val('CO-1').change()
    $('.change-order.modal input[name=amount_cents]').val('$100').change()
    $('.change-order.modal .save').click()

  addLineItem = ->
    view.$('.add-line-item').click()
    $('.line-item.modal input[name=label]').val('Test').change()
    $('.line-item.modal input[name=amount_cents]').val('$300').change()
    $('.line-item.modal .save').click()
{% endcodeblock %}

These are the first steps towards testing the functional behavior of the EstimatePriceBreakdownView. Although this view is actually composed of several subviews, the tests are intended to test the view as a whole to simulate a real user as closely as possible. The goal with functional tests should be to ensure every testable action is covered and accounted for. The term 100% test coverage is often tossed around when talking about how many tests to write. However, that usually applies to unit tests and not *just* integration or functional tests. In the case of these types of test, 100% coverage of all functionality is actually really important! Just to reiterate my point earlier, if you have 100% coverage(actually having 100% isn't realistic but you can get pretty close), you can ship code reliably and safely without wasting time worrying about breaking things.

