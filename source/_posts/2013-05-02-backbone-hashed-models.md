---
layout: default
published: true
comments: true
---

# Indexing a Backbone Collection on Foreign Keys

At Arkad, we had the concept of a DataSeries, which was a model that sat at the crossroads of multiple many-to-many relations. Fortunately, some of this was avoidable by denormalizing the necessary attributes into the DataSeries json via RABL templates, but certain client-side foreign key lookups were unavoidable. My first iteration simply used the proxied underscore 'filter', 'find' and 'where' methods to do the necessary lookups.

    getByMappingIdAndCompanyId: function(mapping_id, company_id) {
      var mapping = mappingsCollection.get(mapping_id);
      return _.filter(mapping.get('data_series'), function(ds) {
        var dataSeries = dataSeriesCollection.get(ds.id);
        return (dataSeries !== undefined && dataSeries.get('company_id') === company_id);
      }, this);
    }
    
The code above worked, but it was horribly inefficient. In order to find a DataSeries by a mapping_id and company_id, you had to first find the mapping and then look through all the DataSeries it was associated with to find the one with the right company_id. Just so we're all on the same page the associations looked like this:

    DataSeries
      belongs_to :company
      has_many :mappings
      
    Mapping
      belongs_to :company
      has_many :data_series
      
    Company
      has_many :data_series
      has_many :mappings
      
For the next iteration, I modified the RABL file to include mapping_ids in the DataSeries models, and data_series_ids in the Mapping models. Client-side I hashed the models at creation via 'add', 'remove', 'reset' events on the collection. 

    _hashModel: function(model) {
      _.each(model.get('mapping_ids'), function(id) {
        if(!_.has(this._mappingMap, id))
          this._mappingMap[id] = {};
        this._mappingMap[id][model.get('company_id')] = model;
      }, this);

      if(!_.has(this._companyMap, model.get('company_id')))
        this._companyMap[model.get('company_id')] = {};
      this._companyMap[model.get('company_id')][model.id] = model;
    },

There was two main scenarios for accessing the models: by both mapping_id and company_id, and just by company_id. So I created a hash to handle each case, in order to keep the lookup time constant. Although this pseudo-map isn't nearly the same as a GLib hash table or STL Map, it's safe to assume that lookups are O(1). If you're going to be doing more lookups than hashing(adding keys is slow), its safe to use these techniques and worth the additional overhead to hold the 'map' in memory. 

More recently, I've come across a similar scenario that involved model lookup by slug rather than id. MongoDB provides some unsightly ID's so I employed the [mongoid_slug](https://github.com/digitalplaywright/mongoid-slug) gem to handle slugging on the model's name attribute. Mongoid-slug provides an overridden 'find' method on the model, but using Backbone with a Single Page App, meant I could either always force a model lookup to the backend, or check the client-side collection first(ideal). However, due to the large number of models potentially present client-side, I wanted to avoid a call to _.find for a matching name attribute. Here's the extended BaseCollection my collection extends https://gist.github.com/micahroberson/5493159

    define [
      'jquery', 'jqueryui', 'underscore', 'backbone'
    ], 
    ($, jqueryui, _, Backbone) ->
    
      class BaseCollection extends Backbone.Collection
 
        initialize: () ->
          @bindHashEvents()
        
        # e.g.
        # hash_keys: [
        #   'subject',
        #   'message',
        #   'buildSlug' # where buildSlug: () -> "#{@get('name').downcase().replace(/\s/g,'-')}"
        # ]
        hash_keys: []
 
        # key is the attribute originally set via hash_keys(e.g. 'name'), lookup_val is the search term (e.g. 'Frank')
        retrieveFromHash: (key, lookup_val) ->
          if !key then return []
      
          @["_#{key}_hash"][lookup_val]
 
        bindHashEvents: (hash_keys) ->
          # Check for hash keys a la delegateEvents paradigm
          if !(hash_keys || (hash_keys = _.result(this, 'hash_keys'))) then return @
 
          _.each hash_keys, (key) =>
            # initialize hash
            @["_#{key}_hash"] = {}
        
            hashSingle = (model) =>
              # first check if key is method defined on the model, otherwise assume its an attribute on model
              hk = if typeof model[key] == 'function' then model[key]() else model.get(key)
              @["_#{key}_hash"][hk] = model unless !hk
 
            @listenTo @, 'add', hashSingle
 
            @listenTo @, 'reset', (collection) =>
              @["_#{key}_hash"] = {}
              _.each(@models, hashSingle, @)
 
            @listenTo @, 'remove', (model) =>
              hk = if typeof model[key] == 'function' then model[key]() else model.get(key)
              delete @["_#{key}_hash"][hk]




