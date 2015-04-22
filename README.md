# lance-rest-client

## Installing

    npm install lance --save

## Simple usage example

```javascript
var lance = require('lance');

var lance = new Lance({
  baseURL: 'http://apiserver:3000',
  rootPath: '/v1'
});

lance.initialize().then(function() {
  lance.fetch('people').then(function(people) {
    for (var personIdx in people) {
      var person = people.collection()[personIdx]; 
      console.log(person.get('name'));
    }
  });
});
```

What's happening under the hood:

* The client is negotiating with `apiserver` on port `3000` Root URL `/v1`
* The metadata retrieved on that endpoint cointains a link that describe a collection of `people`
* `lance.fetch('people')` gets said collection and returns a promise
* The `people` parameter is populated with the instances of people returned by the Lance server

## Mapping resources to classes

In order to map Lance resources to client classes. The server's `Person` class will be mapped
to the `Person` class:

```javascript
var Person = require('./models/person');

var lance = require('lance');

var lance = new Lance({
  baseURL: 'http://apiserver:3000',
  rootPath: '/v1',
  modelMap: {
    Person: Person
  }
});

lance.initialize().then(function() {
  lance.fetch('people').then(function(people) {
    for (var personIdx in people) {
      var person = people.collection()[personIdx]; 
      console.log(person.get('name'), person.skillCount());
    }
  });
});
```

This snippet is assuming that `./models/person` contains a method such as:

```javascript
Person.skillCount = function() {
  return this.meta('skillLevels').totalCount());
};
```

For more details about dealing with collections see section below.

The following example maps all resources in the test:

```javascript
var lance = require('lance');

var lance = new Lance({
  baseURL: 'http://apiserver:3000',
  rootPath: '/v1',
  modelMap: {
    Metadata: Metadata,
    Person: Person,
    PeopleCollection: PeopleCollection,
    Skill: Skill,
    SkillsCollection: SkillsCollection,
    SkillLevel: SkillLevel,
    SkillLevelsCollection: SkillLevelsCollection,
    PersonLevel: PersonLevel,
    PersonLevelsCollection: PersonLevelsCollection
  }
});

lance.initialize().then(function() {
  // Your code here
});
```

## Sending parameters to requests

```javascript
lance.fetch('person', { uuid: '88b4ddfe-e3c1-11e4-8a00-1681e6b88ec1'} ).then(function(person) {
  console.log(person.get('name'));
});
```

What's happening under the hood:

* The client is checking for the link for fetching `person`
* It will then replace the URI template `{uuid}` for `88b4ddfe-e3c1-11e4-8a00-1681e6b88ec1` as per spec
* The response is parsed as a `Person` object (considering that `lance` was initialized with it as
  `modelMap`)

## Updating a resource

```javascript
lance.fetch('person', { uuid: '88b4ddfe-e3c1-11e4-8a00-1681e6b88ec1'} ).then(function(person) {
  console.log('Previous name:', person.get('name'));
  person.set('name', 'New Person Name');
  person.save().then(function(updatedPerson) {
    console.log('Confirmed name:', updatedPerson.get('name'));
  });
});
```

## Removing a resource

```javascript
lance.fetch('person', { uuid: '88b4ddfe-e3c1-11e4-8a00-1681e6b88ec1'} ).then(function(person) {
  person.remove().then(function() {
    console.log('Person deleted');
  });
});
```

## Adding a resource

```javascript
lance.create('people', {
  name: 'Miles Davis',
  email: 'miles.davis@me.com'
}).then(function(person) {
  console.log(person.get('_id'), person.get('name'));
});
```

or, using a `Person` class:

```javascript
var miles = new Person({
  name: 'Miles Davis',
  email: 'miles.davis@me.com'
});

lance.create('people', miles).then(function(person) {
  console.log(person.get('_id'), person.get('name'));
});
```

## Dealing with collections

```javascript
lance.fetch('people').then(function(people) {

  // Prints the length of the current page
  console.log(people.collection().length);

  // Prints total count of entries, current page number
  // and page count
  console.log(people.totalCount());
  console.log(people.currentPage());
  console.log(people.pageCount());
  
  // Fetches new page and populates the page collection
  console.log(people.nextPage().then(function(people) {
    console.log(people.currentPage());
  }));

  // Fetches the previous page and populates the page collection
  console.log(people.prevPage().then(function(people) {
    console.log(people.currentPage());
  }));
});
```

What's happening under the hood:

* The people collection for the current page is behind `collection()`
* `totalCount()` holds the total amount of resource entries behind this collection
* `currentPage()` holds the current page number
* `pageCount()` is the total amount of pages
* `nextPage()` fetches the next page and populates `collection()` with its results
* `prevPage()` fetches the previous page and populates `collection()` with its results

## Fetching a full representation of a resource

As per Lance-REST spec, sometimes a resource might have been represented in an partial manner. In
order to fetch it, let's assume `person` is already a partial representation of a person:

```javascript

// i.e. undefined
console.log(person.get('email'));

person.more().then(function(person) {
  // i.e. miles.davis@me.com
  console.log(person.get('email'));
});
```

## Lance API

### constructor(options)

Creates a lance instance. Options are:

* `baseURL` (mandatory): base URL for the server. i.e. `http://apiserver:3000`
* `rootPath` (mandatory): resource path for Lance's root document. i.e. `/`
* `modelMap` (optional): an object mapping Lance class types identifiers to JavaScript classes (instances
  of BaseModel)

### lance.initialize()

Initializes a lance instance. Returns a promise that is trigered when the client has fully initialized.

### lance.fetch(linkName, data)

Fetches the link specified by `linkName`. Optionally use the object sent as `data` to proceed with
URI template preparation. Returns a promise that is triggered with the object just fetched.

### lance.create(linkName, data)

Creates (`POST`) a resource to `linkName`. Data is an object representing the entity to be created.
Returns a promise that is triggered with the object that was just created.

## Base Model API

### get(field)

Getter for the internal representation of the `field`. 

### set(field, value)

Setter for the internal representation of the `field` with `value`.

### meta(field)

Getter for meta fields (inside `_meta` nodes).

### fetch(linkName)

Similar to `lance.fetch(linkName)` with exception of not having the option of sending data (URI template).
It will return a promise that is triggered when the link represented by `linkName` returns its object.
  
## collection()

If the object is identified as a collection, this method returns the current page's collection. Otherwise
it returns `undefined`.

## totalCount()

If the object is identified as a collection, this method returns the total amount of entries
for this collection beyond the boundaries of the current page (i.e. a `collection().length` might
be `20` while a `totalCount()` might be `400` - in practice it means that the current page has 20
entries but the collection is 400). Otherwise it returns `undefined`.

## currentPage()

If the object is identified as a collection, this method returns the number of the current page
(1-based). Otherwise it returns `undefined`.

## pageCount()

If the object is identified as a collection, this method returns the amount of pages in this collection.
Otherwise it returns `undefined`.

## nextPage()

If the object is identified as a collection, this method returns a promise that will get resolved
with the next page of the collection. Otherwise it returs `undefined`. If there's no next page the
response is `[]`.

## prevPage()

If the object is identified as a collection, this method returns a promise that will get resolved
with the previous page of the collection. Otherwise it returs `undefined`. If there's no next page the
response is `[]`.