---
publish: true
layout: post
title: "Saving Drag and Drop Sort Position"
date: 2013-09-25 10:35
updated: 2013-09-25 10:35
comments: true
keywords: drag and drop, sort position, sort index, efficient, store, save
description: Efficiently store and retrieve sort ordering/positioning in a drag and drop GUI built with jQuery UI, Backbone.js, Rails and MongoDB.
cover: http://micahroberson.com/images/dragndrop.png
---

We recently began work on an in-house solution for managing our projects at [ReadyApps](https://www.readyappsdev.com), and this solution requires that we have the ability to drag and drop 'cards' in the same light as Trello or Blossom. Drag and drop interfaces are certainly nothing new and neither is rearranging the position of an element in a sorted array. However, I was unable to find much information on doing this in the most efficient way possible from a client-side javascript application. 
<!--more-->
{% img width-100 /images/dragndrop.png %}

I considered the following factors/constraints when determining the best implementation:

- Update new position(s) with a single ajax call
- Allow for consecutive client-side moves without having to redraw the entire list after each move
- Minimize backend work to save a move
- Minimize backend work to retrieve cards from database and sort into proper order
- Allow for real-time client-side updates from other users (ie. card position changes via websockets)
- There should be no limit to the number of times a user can rearrage a set of cards
- The solution should allow for a sufficiently high number of cards to be sorted and retrieved in decent amount of time

The simplest solution would be to simply store a 'position' on each card as an integer and sort by that column when retrieving from the database. However, this solution would require that dragging Card A from position 8 to position 1 would need to update all cards with position > 1. Although the query is trivial, these cards would need to be updated client-side as well in order to maintain a clean state. Clean state is important for allowing consecutive moves as well as updates from websockets. 

Another solution is to store the sort order outside of the actual cards - ie. an array of card ids on the project model. I would prefer the order to be saved on the actual card (for API purposes, it makes more sense to be able to sort the cards without having the actual project model as well), this method also requires that we redraw every card on a change. I suppose we could diff the arrays client-side and determine which cards need to be moved, but I haven't actually thought it through enough to be sure that would work efficiently. 

The runner-up solution was to store the position on a card but as a double rather than an integer. Keeping this value a double means we can simply set the new position to half of the difference between the two cards. Moving a card from position 8 to position 1(assuming all cards have integer positions at this point), we would set the new position to (1 - 0) / 2 = 0.5. This would require a single save on the card that changed position, and we could still sort on the position column when retrieving the data. This also allows for handling position changes from other users via websockets. The one limitation is the number of rearrangements a user can make. Eventually we're going to max out the decimal places on the double. We could re-sort all of the elements with a background job when the cards approach this limit, but that doesn't seem like the right way to solve the problem. 

The solution we ended up going with is to simulate a linked-list in order to maintain relevent position. Each card stores the id of the card it follows. This makes client-side updates extremely easy: if a card model changes, we simply find the id it follows and append it right after. As far as the client knows, only a single ajax call is required. Server-side, there's actually two more operations that need to take place: updating the new follower of the card that moved, and updating the previous follower of the card that moved. When retrieving the data server-side, we can put all of the cards in order with a simple loop. Although the operation isn't linear, the complexity is roughly O(n log n) - on par with mergesort. Some quick benchmarking showed the following results:

- 0.001941s for 10 cards
- 0.205871s for 500 cards
- 0.812723s for 1000 cards

Although it starts to climb quickly upwards of 1000 cards, its highly unlikely that a user would ever add that many cards and still expect good performance, so we're content with the results. 

Implementing the solution client-side with [Backbone.js](http://www.backbonejs.org) and [jQuery UI](http://www.jqueryui.com) is done with the following method to be called after a card is 'dropped':
{% codeblock lang:coffeescript %}
  updatePositions: (e, ui) =>
    $ui = ui.item
    id = $ui.data('model-id')
    $after = $ui.prev()

    afterCardId = if $after then $after.data('model-id') else null

    card = @cardsCollection.get(id)
    card.save(
      after_card_id: afterCardId
      )
{% endcodeblock %}
Server-side we'll add a method to be called after updating the after_card_id(ID of the preceding card, nil if first) of the card that moved:

{% codeblock lang:ruby %}
  def update_after_card(stage_id_was, after_card_id_was)
    # Update card that previously came after
    Card.where(after_card_id: id).update_all(
      after_card_id: after_card_id_was
      )

    # Update card that now comes after
    Card.where(:id.ne => id).where(after_card_id: after_card_id).update_all(
      after_card_id: id
      )
  end
{% endcodeblock %}

Also server-side we can retrieve the data from a MongoDB backed Rails app as follows (this was a very quick implementation and there are definitely some improvements that could be made to improve the efficiency):

{% codeblock lang:ruby %}
  has_many :cards
  ...
  def ordered_cards
    cardsArray = []
    initialCardsArray = cards.to_a
    
    cardIdsArray = []
    indexHash = {}
    i = 0
    x = 0

    if initialCardsArray.size <= 1
      return initialCardsArray
    end

    while initialCardsArray.size > 0 do
      card = initialCardsArray[i]
      index = cardIdsArray.index(card.after_card_id)
      
      if card.after_card_id.nil?
        cardsArray.insert(0, card)
        cardIdsArray.insert(0, card.id.to_s)
        initialCardsArray.delete(card)
      elsif !index.nil?
        pos = index + 1
        cardsArray.insert(pos, card)
        cardIdsArray.insert(pos, card.id.to_s)
        initialCardsArray.delete(card)
      else
        i += 1
      end
      
      if i == initialCardsArray.size
        i = 0
      end
      
    end

    cardsArray
  end
{% endcodeblock %}

Any comments or additional solutions would be greatly appreciated! Comments section coming soon!
