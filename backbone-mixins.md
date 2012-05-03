Mixins in Backbone
==================

In our Backbone.js code we have several times seen the need to include
similar methods in models, collections and views. I've seen many create
a component which they extend from, e.g. a `PaginationCollection`, but
we didn't like this solution. We solved it by introducing mixins.

What's a mixin?
---------------

A mixin allows you to extend your Backbone components with utility
functions. Let's, as an example, say that we have pagination which is
similar in many views, but not in all. It might also be that some views
don't want to paginate a collection at all. How should we include this
functionality?

We have the following API for adding extra functionality:

```javascript
UserView.mixin(Pagination);
AppView.mixin(Transitions);
```

What's cool about our implementation is that we can even include our own
`initialize`, `render` and `events` in these view mixins. They only end
up extending the existing functions. Thus, with these in place you can
(almost always) fully contain functionality in the mixin.

Implementation
--------------

Let's have a look at how these mixins can be implemented. First, we can
see the basic function which implements mixins for views:

```javascript
Utils = {};
Utils.mixin = function(from) {
  var to = this.prototype;

  _.defaults(to, from);
  _.defaults(to.events, from.events);

  Utils.extendMethod(to, from, "initialize");
  Utils.extendMethod(to, from, "render");
};
```

Helper method:

```javascript
Utils.extendMethod = function(to, from, methodName) {
  if (!_.isUndefined(from[methodName])) {
    var old = to[methodName];
    to[methodName] = function() {
      var oldReturn = old.apply(this, arguments);
      from[methodName].apply(this, arguments);
      return oldReturn;
    };
  }
};
```

Now we need to include this mixin in such a way that `this` means the
correct thing and we can use it like we saw in the example above. This
can be done in two ways, both of which utilize the `mixin` method we
created above:

1. As [mentioned](http://documentcloud.github.com/backbone/#FAQ-extending)
   in the Backbone docs it's okey to add methods directly to a Backbone
   component. We can use this idea and add `mixin` like this:

   ```javascript
   Backbone.View.mixin = Utils.mixin;
   ```
2. If you create a layered architecture, you can include `mixin` in one
   of your layers. We created a `BaseView` which we created all our
   views from. Remember, this is how Backbone views are defined:

   ```javascript
   Backbone.View.extend(properties, [classProperties])
   ```

   So, in order to create our wanted API we can add `mixin` as a class
   property on our `BaseView`. This can for example be implemented as
   follows:

   ```javascript
   var BaseView = Backbone.View.extend({
     // lots of methods
   }, {
     mixin: Utils.mixin
   })
   ```

Both of these will end up giving us access to `mixin` on our created
views. Let's look at a complete implementation based on the latter:

```javascript
var BaseView = Backbone.View.extend({
    // add base methods
}, {
    mixin: Utils.mixin
});

var UserView = BaseView.extend({
    events: {
        "click h1": "user"
    },

    initialize: function() {
        console.log("user init");
    },

    user: function() {
        return "user";
    }
});

var Pagination = {
    events: {
        "click a.next": "paginate"
    },

    initialize: function() {
        console.log("pagination init");
    },

    next: function() {
        return "next for: " + this.user();
    },

    paginate: function() {
        console.log("paginating");
    }
};

UserView.mixin(Pagination);

var view = new UserView();
// this initialization console logs (in order):
// "user init"
// "pagination init"

console.log(view.user()); // "user"
console.log(view.next()); // "next for: user"
console.log(view.events); // {
                          //   "click a.next": "paginate",
                          //   "click h1": "user"
                          // }
```

Testing
-------

We could have tested the mixins by themself, but as they are always used
by other components, that felt strange. Many of them create essential
functionality in the components they are mixed with, and many components
also have their idiosyncracies in how they use the mixed in code.

Luckily, Jasmine has a great way to include similar tests several
places.

TODO: Example

---

This blog post, and our solution to the mixin problem, was heavily
influenced by Dmitry Polushkin's [gist on
mixins](https://gist.github.com/1256695).