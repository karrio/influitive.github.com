---
  title: Persisting has_many associations on a Rails service with Backbone Collections
  author: Brad
  lead: "Adding and deleting model associations requires a bit of legwork on the client side.  See how this is simplified with [Backbone.RailsNestedAttributesCollection](https://github.com/bradrobertson/RailsNestedAttributesCollection)"
  layout: post
---

While building a [Backbone.js](http://documentcloud.github.com/backbone/) frontend for a new section of the [AdvocateHub](http://influitive.com/sign_up.html), we quickly noticed a pattern evolving regarding associated models and rails nested attributes.  Here at Influitive we make used of `accepts_nested_attributes_for` quite often which makes it incredibly simple to perform CRUD on associated models.  To delete an associated model one just sets a `_destroy` parameter of that model to true and the rest is taken care of on save.  Unfortunately, this means we can't just delete these associated models willy nilly from our collections on the client side as the deletion will never get sent back to the server.

It may be fine to leave deleted these models interspersed in the main collection, but problems start to arise if you use sorting or positioning.  Under most circumstances one wouldn't want a deleted model to be sorted and positioned amongst the models we want to keep. One could alleviate this issue somewhat by always filtering the collection by models that aren't `_destroy`'d, but it quickly becomes tedious.

After enduring this tedium for a short period, it quickly became obvious to us that we were better dealing with two collections on the client side: one for the destroyed models, and one for the other new or existing models that required persisting.  Enter [Backbone.RailsNestedAttributesCollection](https://github.com/bradrobertson/RailsNestedAttributesCollection).  This is a pretty simple implementation that subclasses Backbone.Collection to automatically track its deleted models.

Most of the implementation should be fairly obvious at first glance, but I'll give a quick overview of the code and how we use it at Influitive.

The heart of the Collection is the `deleted` attribute that is automatically added to your collection on initialization.  We do this by overriding the "private" `_reset` function on Collection to add in another collection that automatically tracks its parent.

    _reset: function(){
      Backbone.Collection.prototype._reset.call(this);

      // ...

      var ChildCollection = DeletedCollection.extend({parent: this});
      this.deleted = new ChildCollection();
    }

Our `DeletedCollection` gets passed a reference to `this` which is our main collection.  The `DeletedCollection` then binds to the `remove` event of the parent collection and uses it to maintain its own list:

    var DeletedCollection = Backbone.Collection.extend({
      initialize: function(){
        _.bindAll(this, "_addDeleted");

        this.parent.bind("remove", this._addDeleted);
      },

      _addDeleted: function(model){
        if( !model.isNew() ){
          model.set({"_destroy": true});
          this.add(model);
        }
      }
    });

It is only concerned with models that were previously persisted, any new models that are deleted from the main collection are obviously not of interest to us.

From there, all we had to do was override the `toJSON` method of our collection to add the deleted collection in and Bob's your Uncle:

    toJSON: function(){
      var models        = Backbone.Collection.prototype.toJSON.call(this),
          deletedModels = Backbone.Collection.prototype.toJSON.call(this.deleted);

      return models.concat(deletedModels);
    }

Here's a full implementation that shows how we achieve persistence using the `Backbone.RailsNestedAttributesCollection`.

    # Ruby
    class Book < ActiveRecord::Base
      has_many :pages
      accepts_nested_attributes_for :pages, :allow_destroy => true
    end

    class Page < ActiveRecord::Base
      belongs_to :book
    end

    // Javascript
    var Book = Backbone.Model.extend({
      initialize: function(){
        this.pages = new PagesCollection( this.get('pages' ) );
      },

      toJSON: function(){
        var attrs = _.clone(this.attributes);
        attrs['pages_attributes'] = this.pages.toJSON();
        // below pages attr might exist from bootstrapping a book model
        // this would fail server side as the `pages=` method expects ruby models
        delete attrs['pages'];

        return attrs;
      }
    });

    var Page = Backbone.Model.extend();

    var PagesCollection = Backbone.RailsNestedAttributesCollection.extend({
      model: Page,
    });

In the end it doesn't really look much different from any other implementation that deals with associated models.  The tracking of deleted models is nicely abstracted away by the new collection.

Enjoy!