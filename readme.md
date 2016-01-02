# Important Notice

This is no longer maintained. Please use [mongoosastic](https://github.com/mongoosastic/mongoosastic) instead.

#`lmongo`

##The Power of Elasticsearch for Mongoose.

`lmongo` is a [mongoose](http://mongoosejs.com/) plugin that integrates your data with [Elasticsearch](http://www.elasticsearch.org), to give you the full power of highly available, distributed search across your data.

`lmongo` is a fork of [elmongo](https://github.com/usesold/elmongo). It is compatible with Elastic Search 1.x and Heroku Bonsai add-on. It also provides more flexible `where` clause options.

If you have [homebrew](http://brew.sh/), you can install and run Elasticsearch with this one-liner:

```
brew install elasticsearch && elasticsearch
```

# Migration from 0.1.x

## Pagination
version 0.1.x
```js
// Paginate through the data
Cat.search({ query: '*', page: 2, pageSize: 25 }, function (err, results) {
  // ...
})
```
new versions
```js
// Paginate through the data
Cat.search({ query: '*', from: 25, size: 25 }, function (err, results) {
  // ...
})
```
> Note: consistent with [pagination for Elastic Search](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/pagination.html)


#Install

```
npm install lmongo
```

#Usage
```js
var mongoose = require('mongoose'),
    lmongo = require('lmongo'),
    Schema = mongoose.Schema

var CatSchema = new Schema({
    name: String
})

// add the lmongo plugin to your collection
CatSchema.plugin(lmongo)

var Cat = mongoose.model('Cat', CatSchema)
```

Now setup the search index with your data:
```js
Cat.sync(function (err, numSynced) {
  // all cats are now searchable in elasticsearch
  console.log('number of cats synced:', numSynced)
})
```

At this point your Cat schema has all the power of Elasticsearch. Here's how you can search on the model:
```js
Cat.search({ query: 'simba' }, function (err, results) {
 	console.log('search results', results)
})

// Perform a fuzzy search
Cat.search({ query: 'Sphinxx', fuzziness: 0.5 }, function (err, results) {
	// ...
})

// Search in specific fields
Cat.search({ query: 'Siameez', fuzziness: 0.5, fields: [ 'breed'] }, function (err, results) {
    // ...
})

// Paginate through the data
Cat.search({ query: '*', from: 0, size: 25 }, function (err, results) {
 	// ...
})

// Use `where` clauses to filter the data
Cat.search({ query: 'john', where: { age: 25, breed: 'siamese' } }, function (err, results) {
	// ...
})

// The `where` clauses support `and` and `or`
Cat.search({ query: 'john', where: { and: [{or: [{age: 25}, {breed: 'siamese'}]}, {age:30}] }, function (err, results) {
  // ...
})

// The `where` clause has full support for raw elastic search filters via `$raw`
// You can also mix the high-level lmonogo filter and the low-level elastic search filter together
Cat.search({ query: 'john', where: { and: [{age:30}, {'$raw': {not: {ids: ["53de7747bb8b38f39b7efe28"]}}}] }, function (err, results) {
  // ...
})
```

> To use raw elastic search filter, please refer to [Elastic Search Filter Guide](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-filters.html)

After the initial `.sync()`, any **Cat** models you create/edit/delete with mongoose will be up-to-date in Elasticsearch. Also, `lmongo` reindexes with zero downtime. This means that your data will always be available in Elasticsearch even if you're in the middle of reindexing.

#API

##`Model.sync(callback)`

Re-indexes your collection's data in Elasticsearch. After the first `.sync()` call, Elasticsearch will be all setup with your collection's data. You can re-index your data anytime using this function. Re-indexing is done with zero downtime, so you can keep making search queries even while `.sync()` is running, and your existing data will be searchable.

Example:
```js
Cat.sync(function (err, numSynced) {
	// all existing data in the `cats` collection is searchable now
    console.log('number of docs synced:', numSynced)
})
```

##`Model.search(searchOptions, callback)`

Perform a search query on your model. Any values you provide will override the default search options. The default options are:

```js
{
    query: '*',
    fields: [ '_all' ],	// searches all fields by default
    fuzziness: 0.0,		// exact match by default
    size: 25,
    from: 0
}
```

##`Model.plugin(lmongo[, options])`

Gives your collection `.search()` and `.sync()` methods, and keeps Elasticsearch up-to-date with your data when you insert/edit/delete documents with mongoose. Takes an optional `options` object to tell `lmongo` the url that Elasticsearch is running at. In `options` you can specify:

 * `url` - the url that Elasticsearch is running on. If this is not passed, then the following options are used.
 * `host` - the host that Elasticsearch is running on (defaults to `localhost`)
 * `port` - the port that Elasticsearch is listening on (defaults to `9200`)
 * `prefix` - adds a prefix to the model's search index, allowing you to have separate indices for the same collection on an Elasticsearch instance (defaults to no prefix)

Suppose you have a test database and a development database both storing models in the `Cats` collection, but you want them to share one Elasticsearch instance. With the `prefix` option, you can separate out the indices used by `lmongo` to store your data for test and development.

For tests, you could do something like:
 ```js
Cat.plugin(lmongo, { host: 'localhost', port: 9200, prefix: 'test' })
 ```
And for development you could do something like:
```js
Cat.plugin(lmongo, { host: 'localhost', port: 9200, prefix: 'development' })
```

This way, you can use the same `mongoose` collections for test and for development, and you will have separate search indices for them (so you won't have situations like test data showing up in development search results).

**Note**: there is no need to specify a `prefix` if you are using separate Elasticsearch hosts or ports. The `prefix` is simply for cases where you are sharing a single Elasticsearch instance for multiple codebases.

##`lmongo.search(searchOptions, callback)`

You can use this function to make searches that are not limited to a specific collection. Use this to search across one or several collections at the same time (without making multiple roundtrips to Elasticsearch). The default options are the same as for `Model.search()`, with one extra key: `collections`. It defaults to searching all collections, but you can specify an array of collections to search on.

```js
lmongo.search({ collections: [ 'cats', 'dogs' ], query: '*' }, function (err, results) {
	// ...
})
```

By default, `lmongo.search()` will use `localhost:9200` (the default Elasticsearch configuration). To configure it to use a different url, use `lmongo.search.config(options)`.

##`lmongo.search.config(options)`

Configure the Elasticsearch url that `lmongo` uses to perform a search when `lmongo.search()` is used. `options` can specify the same keys as `Model.plugin(lmongo, options)`. `lmongo.search.config()` has no effect on the configuration for individual collections - to configure the url for collections, use `Model.plugin()`.

Example:
```js
lmongo.search.config({ host: something.com, port: 9300 })
```

#Autocomplete

To add autocomplete functionality to your models, specify which fields you want autocomplete on in the schema:
```js
var CatSchema = new Schema({
    name: { type: String, autocomplete: true },
    age: { type: Number },
    owner: { type: ObjectId, ref: 'Person' },
    nicknames: [ { type: String, autocomplete: true } ]
})

// add the lmongo plugin to your collection
CatSchema.plugin(lmongo)

var Cat = mongoose.model('Cat', CatSchema)

var kitty = new Cat({ name: 'simba' }).save()
```

Setup the search index using `.sync()`:
```js
Cat.sync(function (err, numSynced) {
  // all cats are now searchable in elasticsearch
  console.log('number of cats synced:', numSynced)
})
```

Now you have autocomplete on `name` and `nicknames` whenever you search on those fields:
```js
Cat.search({ query: 'si', fields: [ 'name' ] }, function (err, searchResults) {
    // any cats having a name starting with 'si' will show up in the search results
})
```

-------

## Running the tests

```
npm test
```

-------

## License

(The MIT License)

Copyright (c) by Sold. <tolga@usesold.com>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
