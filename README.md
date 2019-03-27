# tunguska-reactive-aggregate

Reactively publish aggregations.

    meteor add tunguska:reactive-aggregate

This helper can be used to reactively publish the results of an aggregation.

Originally based on `jcbernack:reactive-aggregate`.

This clone removes the dependency on `meteorhacks:reactive-aggregate` and instead uses the underlying MongoDB Nodejs library. In addition, it uses ES6/7 coding, including `async` and `await` and `import/export` syntax, so should be `import`ed into your (server) codebase where it's needed.

In spite of those changes, the API is basically unchanged. However, there are a few additional properties of the `options` parameter. See the notes in the **Usage** section.

## Usage

```js
import { ReactiveAggregate } from 'meteor/tunguska:reactive-aggregate';

Meteor.publish('nameOfPublication', function() {
  ReactiveAggregate(sub, collection, pipeline, options);
});
```

- `sub` should always be `this` in a publication.
- `collection` is the Mongo.Collection instance to query, or `null`. Setting `collection` to `null` disables backwards compatibility, in which an observer is automatically added on the collection.

  The backwards-compatible, deprecated options `observeSelector` and `observeOptions` will continue to be honoured when `collection` is a `Mongo.Collection`, but the recommended approach is to use `null` and `options.observers`, which replaces the original method with something more versatile. There is no guarantee that deprecated options will continue to be honoured in future releases.

- `pipeline` is the aggregation pipeline to execute.
- `options` provides further options:

    The recommended approach is to set this option to `false` and ensure the required cursor is specified in the `observers` array.
  
  - `aggregationOptions` can be used to add further, aggregation-specific options. See [standard aggregation options](http://mongodb.github.io/node-mongodb-native/3.1/api/Collection.html#aggregate) for more information.
  - `clientCollection` defaults to the same name as the original collection, but can be overridden to send the results to a differently named client-side collection.
  - `observers`: An array of cursors. Each cursor is the result of a `Collection.find()`. Each of the supplied cursors will have an observer attached, so any change detected (based on the selection criteria in the `find`) will re-run the aggregation pipeline.
  - `debounceCount`: An integer representing the number of observer changes across all observers before the aggregation will be re-run. Defaults to 100. Used in conjunction with `debounceDelay` to fine-tune reactivity.
  - `debounceDelay`: An integer representing the maximum number of milli-seconds to wait for observer changes before the aggregation is re-run. Defaults to 100. Used in conjunction with `debounceCount` to fine-tune reactivity.

  :hand: The following parameters are **deprecated** and will be removed in a later version. Both these parameters are now effectively absorbed into the `observers` option and if required should be added as a cursor (or another cursor) to the array of cursors in that. Setting either of these to anything other than the empty object `{}` will result in a deprecation notice to the server console.
  - ~~`observeSelector` can be given to improve efficiency. This selector is used for observing the collection.
  (e.g. `{ authorId: { $exists: 1 } }`)~~
  - ~~`observeOptions` can be given to limit fields, further improving efficiency. Ideally used to limit fields on your query.
  If none are given any change to the collection will cause the aggregation to be re-evaluated.
  (e.g. `{ limit: 10, sort: { createdAt: -1 } }`)~~

## Quick Example

A publication for one of the
[examples](https://docs.mongodb.org/v3.0/reference/operator/aggregation/group/#group-documents-by-author)
in the MongoDB docs would look like this:

```js
Meteor.publish("booksByAuthor", function () {
  ReactiveAggregate(this, Books, [{
    $group: {
      _id: "$author",
      books: { $push: "$$ROOT" }
    }
  }]);
});
```

## Extended Example

Define the parent collection you want to run an aggregation on. Let's say:

```js
import { Mongo } from 'meteor/mongo';
export const Reports = new Mongo.Collection('Reports');
```

...in a location where all your other collections are defined, say `/imports/both/Reports.js`

Next, prepare to publish the aggregation on the `Reports` collection into another client-side-only collection we'll call `clientReport`.

Create the `clientReport` in the client side (it's needed only for client use). This collection will be the destination into which the aggregation will be put upon completion.

Publish the aggregation on the server:

```js
Meteor.publish("reportTotals", function() {
  // Remember, ReactiveAggregate doesn't return anything
  ReactiveAggregate(this, Reports, [{
    // assuming our Reports collection have the fields: hours, books
    $group: {
      '_id': this.userId,
      'hours': {
      // In this case, we're running summation.
        $sum: '$hours'
      },
      'books': {
        $sum: 'books'
      }
    }
  }, {
    $project: {
      // an id can be added here, but when omitted,
      // it is created automatically on the fly for you
      hours: '$hours',
      books: '$books'
    } // Send the aggregation to the 'clientReport' collection available for client use by using the clientCollection property of options.
  }], { clientCollection: 'clientReport' });
});
```

Subscribe to the above publication on the client:

```js
import { Mongo } from 'meteor/mongo';

// Define a named, client-only collection, matching the publication's clientCollection.
const clientReport = new Mongo.Collection('clientReport');

Template.statsBrief.onCreated(function() {
  // subscribe to the aggregation
  this.subscribe('reportTotals');

// Then in our Template helper:

Template.statsBrief.helpers({
  reportTotals() {
    return clientReport.find();
  },
});
```

Finally, in your template:

```handlebars
{{#each report in reportTotals}}
  <div>Total Hours: {{report.hours}}</div>
  <div>Total Books: {{report.books}}</div>
{{/each}}
```

Your aggregated values will therefore be available in the client and behave reactively just as you'd expect.

## Using `$lookup`

The use of `$lookup` in an aggregation pipeline introduces the possibility that the aggregation pipeline will need to be run when any or all of the collections change.

By default, only the base collection is observed for changes. However, it's possible to specify an arbitrary number of observers on disparate collections. In fact, it's possible to observe a collection which is not part of the aggregation pipeline to trigger a re-run of the aggregation. This introduces some interesting approaches towards optimising "heavy" pipelines on very active collections.

```js
Meteor.publish("biographiesByWelshAuthors", function () {
  ReactiveAggregate(this, null, [{
    $lookup: {
      from: "books",
      localField: "_id",
      foreignField: "author_id",
      as: "author_books"
    }
  }], {
    observers: [
      Authors.find({ nationality: 'welsh'}),
      Books.find({ category: 'biography' })
    ]
  });
});
```

The aggregation will re-run whenever there is a change to the "welsh" authors in the `authors` collection or if there is a change to the biographies in the `books` collection.

No `debounce` parameters have been specified, so any changes will only be made available to the client when the 100 changes have been seen across both collections, or after 100mS, whichever occurs first.

---

Enjoy aggregating `reactively`!
