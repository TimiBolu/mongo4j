# Mongo4J
[![Build Status](https://travis-ci.org/SvenWesterlaken/mongo4j.svg?branch=master)](https://travis-ci.org/SvenWesterlaken/mongo4j)
[![Coverage Status](https://coveralls.io/repos/github/SvenWesterlaken/mongo4j/badge.svg?branch=master)](https://coveralls.io/github/SvenWesterlaken/mongo4j?branch=master)
[![npm](https://img.shields.io/npm/v/mongo4j.svg)](https://www.npmjs.com/package/mongo4j)
[![node](https://img.shields.io/node/v/mongo4j.svg)](https://www.npmjs.com/package/mongo4j)
[![Greenkeeper badge](https://badges.greenkeeper.io/SvenWesterlaken/mongo4j.svg)](https://greenkeeper.io/)
[![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release)
[![npm](https://img.shields.io/npm/dt/mongo4j.svg)](https://www.npmjs.com/package/mongo4j)

[![NPM](https://nodei.co/npm/mongo4j.png)](https://nodei.co/npm/mongo4j/)

> A [mongoose](http://mongoosejs.com/) plugin to automatically maintain nodes in [neo4j](https://neo4j.com/)
>
> _**Currently in development**_

## Table of contents
- [Installation](#installation)
- [Setup](#setup)
  - [Single driver](#single-driver)
  - [Multiple drivers](#multiple-drivers)
  - [Add the plugin to the schema](#add-the-plugin-to-the-schema)
- [Schema configuration options](#schema-configuration-options)
  - [Standard Properties](#standard-properties)
  - [Relationships (References, Nested References & Subdocuments)](#relationships-references-nested-references--subdocuments)
- [Saving](#saving)
- [Updating](#updating)
- [Removing](#removing)
- [Upcoming features & to-do-list](#upcoming-features--to-do-list)
- [Credits](#credits)

## Why Mongo4j?

The usage of mongo4j is found in the term [polyglot persistence](https://en.wikipedia.org/wiki/Polyglot_persistence). In this case you will most likely want to combine the 'relationship-navigation'
of neo4j while still maintaining documents in mongodb for quick access and saving all information. Unfortunately this also brings in extra maintenance to keep both databases in-sync. For this matter, several plugins and programs have been written, under which [moneo](https://github.com/srfrnk/moneo), [neo4j-doc-manager](https://neo4j.com/developer/neo4j-doc-manager/) & [neomongoose](https://www.npmjs.com/package/neomongoose).

These are great solutions, but I've found myself not fully satisfied by these. The doc manager, for example, needs another application layer to install and run it. The other two solutions were either out of date or needed a manual form of maintaining the graphs in neo4j. That's why I decided to give my own ideas a shot in the form of a mongoose plugin.

Although mongo4j doesn't cover changes outside the mongoose context [_yet_](https://docs.mongodb.com/manual/changeStreams/), it still automatically updates, removes and adds graphs according to the given schema configuration. In addition to this it adds extra functions to access the graphs from the models through mongoose. This way, there is no need to keep two different approaches to the neo4j-database.

## Installation

Download and install the package with npm:
```bash
npm install -save mongo4j
```

## Setup

Before you use (require) mongo4j anywhere. **First initialize it with drivers.**

This creates the singleton pattern lifecycle of driver(s) stated by the [neo4j-driver library](https://github.com/neo4j/neo4j-javascript-driver#usage-examples).

Same options can be used as the official driver and there is the possibility of initializing multiple drivers in the beginning. Which should be **only one driver per neo4j database**.

Options can be found on the neo4j driver [documentation](https://neo4j.com/docs/api/javascript-driver/current/function/index.html#static-function-driver).

### Single driver

#### mongo4j.init(host, auth, options)
- `host` - Url to neo4j database. Defaults to `bolt://127.0.0.1`
- `auth` - Authentication parameters:
  - `user` - User for neo4j authentication. Defaults to `neo4j`
  - `pass` - Password for neo4j authentication. Defaults to `neo4j`
- `options` - Options for neo4j driver. These can be found in the [documentation](https://neo4j.com/docs/api/javascript-driver/current/function/index.html#static-function-driver).

```javascript
const mongo4j = require('mongo4j');

mongo4j.init('bolt://localhost', {user: 'neo4j', pass: 'neo4j'});
```

### Multiple drivers

#### mongo4j.init(hosts, auth, options)

- `hosts` - Array of hosts. A host in this case consists of:
  - `name` - Identifier to reference this specific driver. _(Must be a string)_ **Required**
  - `url` - Url to neo4j database. Defaults to `bolt://127.0.0.1`
  - `auth` - Authentication parameters:
    - `user` - User for neo4j authentication. Defaults to `neo4j`
    - `pass` - Password for neo4j authentication. Defaults to `neo4j`
  - `options` - Options for neo4j driver. These can be found in the [documentation](https://neo4j.com/docs/api/javascript-driver/current/function/index.html#static-function-driver).
- `auth` - Authentication parameters. _Will be overwritten by individual authentication set in hosts_:
  - `user` - User for neo4j authentication. Defaults to `neo4j`
  - `pass` - Password for neo4j authentication. Defaults to `neo4j`
- `options` - Options for neo4j driver. These can be found in the [documentation](https://neo4j.com/docs/api/javascript-driver/current/function/index.html#static-function-driver). _Will be overwritten by individual options set in hosts_

In the case of multiple drivers make sure you initialize every driver with an identifier (name) in string format for later re-use, otherwise an error will be thrown.

```javascript
const mongo4j = require('mongo4j');

mongo4j.init(
  [{
    name: 'testconnection1',
    url: 'bolt://127.0.0.1',
    auth: {
      user: 'neo4j',
      pass: 'neo4j'
    }
  }, {
    name: 'testconnection2',
    url: 'bolt://127.0.0.1'
  }]
);
```

Authentication can be specified as an second argument to use the same authentication for all drivers. Authentication set per host will override these global authentication settings.

The same goes for options. If you only want to use shared options, make sure you pass `null` as a second argument:

```javascript
const mongo4j = require('mongo4j');

mongo4j.init([host1, host2], null, {connectionPoolSize: 100});
```

### Add the plugin to the schema

#### CustomSchema.plugin(moneo.plugin(identifier))
- `identifier` - Identifier to reference the specific driver to use _(in case of multiple drivers)_

```javascript
// Use the default driver connection (in case of one driver)
PersonSchema.plugin(mongo4j.plugin());

// Use the 'testconnection1' driver to connect to neo4j
PersonSchema.plugin(mongo4j.plugin('testconnection1'))
```

## Schema configuration options

After you have added mongo4j as a plugin to your documentschema there are several properties to configure which and how data of the document is saved in neo4j.

### Standard Properties
These options apply to simple schema properties.

#### neo_prop: `Boolean`
- Defaults to `false`.
- If set to `true` this property will be saved in neo4j.
- **Note:** the `_id` property in mongodb will automatically be added as `m_id` in neo4j.

```javascript
// Save firstName as a property in neo4j
const PersonSchema = new Schema({
  firstName: {
    type: String,
    neo_prop: true
  }
});
```

### Relationships (References, Nested References & Subdocuments)
References, nested references & subdocuments are automatically saved as different nodes as explained [here](#saving).
Therefore there are several options to configure how to relationship is saved.

#### neo_rel_name: `String`
- Defaults to `[PROPERTY NAME]_[DOCUMENT TYPE]_[RELATED DOCUMENT TYPE]`. ie: `SUPERVISOR_CLASS_PERSON`
- **Note:** relationships will be converted to uppercase to conform to the neo4j naming conventions

```javascript
// Results in 'TAUGHT_BY' relationship
const ClassSchema = new Schema({
  teacher: {
    type: mongoose.Schema.ObjectId,
    ref: 'Person',
    neo_rel_name: "Taught By"
  }
});

// Results in 'SUPERVISOR_CLASS_PERSON' relationship (including a start_date property)
const ClassSchema = new Schema({
  supervisor: {
    person: {
      type: mongoose.Schema.ObjectId,
      ref: 'Person'
    },
    start_date: Date
  },
});
```

#### neo_omit_rel: `Boolean`
- Defaults to `false`.
- If set to `true` this relationship will not be saved (omitted) in neo4j.

```javascript
// Don't save the relationship to teacher in neo4j (the teacher can still be saved separately)
const ClassSchema = new Schema({
  teacher: {
    type: mongoose.Schema.ObjectId,
    ref: 'Person',
    neo_omit_rel: true
  }
});
```

## Saving

Saving a mongo-document in neo4j is executed as you would normally. Therefore, return values will still be the same as without mongo4j. Post hooks of `Document.save()` & `Model.insertMany()` will cause the saved document(s) to be saved in neo4j as well.

**Note:** The hooks for saving in neo4j are executed asynchronously.

```javascript
const Person = require('path/to/models/person');

neil = new Person({
  firstName: "Neil",
  lastName: "Young",
  address: {
    city: "Brentwood",
    street: "Barleau St."
  }
});

// Save 'neil' as a node in neo4j (as well as mongodb) according to the schema configuration
neil.save();

const henry = new Person({firstName: "Henry", lastName: "McCoverty"});
const daniel = new Person({firstName: "Daniel", lastName: "Durval"});
const jason = new Person({firstName: "Jason", lastName: "Campbell"});

// Save all three persons in neo4j as well as mongodb
Person.insertMany([daniel, jason, henry]);
```

## Updating

Unfortunately, mongoose doesn't supply a direct way of accessing data in update hooks. Therefore a custom method on the document will be used that will both handle the saving in mongodb and neo4j. It can be seen as wrapper around the original `Document.updateOne()` method.

#### Document.updateNeo(criteria, options, cb)
- **Note**: parameters are identical to that of `Model.updateOne()`. Detailed documentation can therefore be found [here](https://mongoosejs.com/docs/api.html#document_Document-updateOne).
- `criteria`: Data that should be changed _(json format)_
- `options`: options for the `updateOne()` method executed. Refer to the [documentation](https://mongoosejs.com/docs/api.html#query_Query-setOptions) of mongoose for available options.
- `cb`: Callback function to be executed by the `updateOne()` method.

**Returns:** a promise with a result of an array containing (in order):
- Result of the updateOne method. See [documentation](https://mongoosejs.com/docs/api.html#document_Document-updateOne)
- Result of the cypher update query
- Result of the cypher query that deleted all the previous relationships. **(If not executed this will be null)**. Why this query is executed is explained [here](#upcoming-features--to-do-list)

```javascript
// variable `person` refers to a document fetched from the database or returned as a result after saving

// Update the firstname to 'Peter' and lastname to 'Traverson'.
person.updateOne({firstName: 'Peter', lastName: 'Traverson'}).then((results) => {
  // First item of the array is the result of the update query by mongoose
  let mongoUpdateResult = results[0];

  // Second item of the array is the result of the neo4j cypher query for updates
  let neo4jUpdateResult = results[1];

  // Third item of the array is the result of the delete query. In this case null,
  // because the updates didn't involve any changes in relationships between nodes.
  let neo4jDeleteResult = result[2];
});
```


## Removing

Removing a mongo-document in neo4j is executed as you would normally. Post hooks of `Document.remove()` will cause the removed document(s) to be removed in neo4j as well (including subdocuments & relationships).

```javascript
// Remove 'neil' from neo4j as well as mongo
neil.remove()
```

## Upcoming features & to-do-list
This plugin is still in early development so not all features are yet implemented unfortunately.
I'm trying my best to finish these features as fast as possible.
This is also the reason there are only pre-releases yet.

#### To-do-list:

- Wrappers around static functions of a model (adding, updating & deleting)
- **Documentation** (_currently in progress_)
- Code documentation
- Creating a wiki or documentation website
- **Tests** (_currently in progress_)
- Debug Mode (ie. show neo4j query's)
- Helper functions for neo4j access
- State hooks
<!-- - Plugin for subdocuments -->

## Credits

Big shoutout to [srfrnk](https://github.com/srfrnk) for creating the repo called [moneo](https://github.com/srfrnk/moneo).

After some digging through the code, I missed some functionality and saw that the old HTTP driver for neo4j was used.
I decided to rewrite the code with extra functionality and use the [new neo4j driver](https://github.com/neo4j/neo4j-javascript-driver) with _'bolt'_ connection.

Moneo has provided me with the basic info to get started and mongo4j could be seen as a (continued) **version 2.0**.
