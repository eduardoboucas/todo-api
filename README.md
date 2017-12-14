# Building a to-do list application with DADI API

> NOTE: This repository contains the to-do list application already configured, so you can skip the *Installation* section and simply run `npm install && npm start`. The original article can be found on the [DADI Forum](https://forum.dadi.tech/).

In this article, I'll show you how to build a simple application with DADI API, covering everything from the installation process to the definition of the data architecture.

We'll take inspiration from [TodoMVC](http://todomvc.com/), a project that shows how a particular application – a to-do list manager – can be built using different front-end frameworks. Let's build a to-do list application using DADI API.

## Installation

The quickest way to get API up and running is with [DADI CLI](https://docs.dadi.tech/cli). If you haven't already, start by installing it on your system.

```shell
npm install @dadi/cli -g
```

With CLI installed, navigate to the directory where you want to install API and use CLI to start the installation.

```shell
# Choose any directory
cd ~/Sites

# This will install API in a subdirectory called todo-api
dadi api new todo-api
```

The command above will do *a lot*. For starters, it will:

1. Find the latest version of API
2. Pull a blank project (what we call a *boilerplate*) from the [DADI registry](https://github.com/dadi/registry/tree/master/api/boilerplate)
3. Install all the dependencies using NPM

After that, CLI will take you through an interactive setup process where, by asking you a series of questions, it will gather all the information it needs to generate the configuration files API needs.

### Choosing a database engine

The first question you'll get is about which database engine you'd like to use.

```shell
$ dadi api new todo-api
✔ Checking the available versions of DADI API
✔ Pulling the list of available database connectors from NPM
? Which database engine would you like to install? (Use arrow keys)
❯ @dadi/api-mongodb — A MongoDB adapter for DADI API 
  @dadi/api-filestore — A JSON datastore adapter for DADI API
```


When we first launched API, MongoDB was our database engine of choice. This has been the case until version 3.0, released earlier this month, where we rebuilt the application core such that virtually any database engine can be used, as long as there's an adapter for it (or a *data connector*, as we call them).

We're working on a few data connectors for engines like CouchDB and RethinkDB, but don't feel discouraged if your engine of choice isn't covered yet – we're publishing an article later this month that will show how you can easily build your own data connector for DADI API.

For our to-do list app, and to keep things simple, we'll choose `@dadi/api-filestore`, which uses an in-memory file store, which is persisted to disk at regular intervals as JSON files.

As for the rest of the questions, you can customise your installation as you see fit. Here's what I went with:

- *What is the name of this DADI API instance?* `Todos API`
- *What is the IP address the application should run on?* `0.0.0.0`
- *What is the port number?* `8081`
- *What protocol would you like to use?* `HTTP (insecure)`

- *What is the hostname or domain where your API can be accessed at?* `todos-api.com`
- *What is the port?* `80`

- *What is the name of the database?* `todosapi`
- *Where do you want to save your database files?* `workspace/db`

- *Would you like to create a client?* `Yes`
- *What is the client ID?* `todoClient`
- And what is the secret? Press <Enter> if you want us to generate one for you.
- *What level of permissions should this client get?* `Administrator`

- *Would you like to cache items on the local filesystem?* `Yes`
- *What is the path to the cache directory?* `./cache/api`
- *Would you like to cache items on a Redis server?* `No`

- *Where should API store uploaded assets?* `Nowhere, I don't want API to handle media`

- *Which environment does this config apply to?* `Development`

If all goes well, you should see a confirmation message telling you which configuration files were created and where, as well as the commands you must run to spin up your freshly-made API instance.

```shell
✔ API configuration file written to /Users/eduardoboucas/Sites/todo-api/config/config.development.json.
✔ Database configuration file written to /Users/eduardoboucas/Sites/todo-api/config/filestore.development.json.
✔ Created client with ID todoClient and type admin. The secret we generated for you is h5zfpj5gb7 – store it somewhere safe!

All done! Run the following command to launch your new instance of DADI API:

cd todo-api && npm start
```

To confirm that your API is up and running, fire a `GET` request to http://localhost:8081/hello. You should see a message saying `Welcome to API`.

## Authentication

DADI API uses 2-step oAuth to authenticate requests. In the previous step, we created a client ID/secret pair – in my case, these are `todoClient`/`h5zfpj5gb7`. You'll need to exchange your credentials for a bearer token, valid for a configurable period of time, which is what will identify you as a consumer on every request to API.

To get a bearer token, load [Postman](https://www.getpostman.com/) (or any other HTTP client) and fire a `POST` request to the `/token` endpoint in your API.

```
POST http://localhost:8081/token

Accept: application/json
Content-Type: application/json

{
    "clientId": "todoClient",
    "secret: "h5zfpj5gb7
}
```

The response will be a JSON payload containing the bearer token and the amount of time, in seconds, for which the token will be valid.

```json
{
    "accessToken": "2c483e66-74c9-4535-ac7b-26b4ca53ef53",
    "tokenType": "Bearer",
    "expiresIn": 1800
}
```

Keep this bearer token handy, we'll need it in a bit. Now, let's start shaping our data.

## Data architecture

Data is organized in collections, which are declared as JSON schema files. This is where we'll define things like field types and validation rules.

For our to-do list app we'll need two collections. The main one is *items*, where we'll store all our to-do items. To take the TodoMVC concept a step further, let's also add a *users* collection, which we'll use to assign items to people.

Collections live in `workspace/collections`. Because API has first-class support for endpoint versioning, you'll find a `1.0` directory created for you. If you were to release a new version of your endpoints, you'd create a new version (say `2.0`) whilst retaining the old, deprecated `1.0` until you decide to decomission it.

Under each version directory you'll find a subdirectory for each database, in case you decide to split collections into different databases.

Let's create our users collection. Under `workspace/collections/1.0/todos`, create a `collection.users.json` file with the following: 

```json
{
  "fields": {
    "name": {
      "type": "String"
    },
    "email": {
      "type": "String",
      "validation": {
        "regex": {
          "pattern": "^[A-Z0-9._%+-]+@[A-Z0-9.-]+\\.[A-Z]{2,}$",
          "flags": "gi"
        }
      }
    }
  },
  "settings": {
    "description": "Represents people who have registered with the system"
  }
}
```

Let's take a closer look at what's happening here. The `fields` block defines the fields and their types and constraints.

We're defining a `name` and `email` fields, both strings, with the latter having to meet a specific validation rule, using a regular expression that defines what represents a valid email address.

We can now move to the *items* collection. Create a `collection.items.json` in the same directory and paste the following:

```json
{
  "fields": {
    "name": {
      "type": "String",
      "required": true
    },
    "assignee": {
      "type": "Reference",
      "settings": {
        "collection": "users"
      }
    },
    "completed": {
      "type": "Boolean",
      "default": false
    }
  },
  "settings": {
    "description": "Represents to-do items"
  }
}
```

If the collection schema for users made sense to you, this one shouldn't feel too dissimilar. The only particularity we see is a field of type `Reference`, which will link users to to-do items. You can look at this as a 1-N relationship in more traditional database terms.

## Adding data

Time to add some data! Let's start by creating a to-do item by firing a `POST` request to the *items* collection endpoint.

```
POST http://localhost:8081/1.0/todos/items

Accept: application/json
Authorization: Bearer 2c483e66-74c9-4535-ac7b-26b4ca53ef53
Content-Type: application/json

{
  "name": "Build a RESTful API with DADI API",
  "completed": false
}
```

Note the `Authorization` header containing the bearer token we got earlier. The response will be a `results` array containing the documents inserted.

A series of internal fields (prefixed with an underscore) will be added, like a unique document identifier and the creation date.

```json
{
  "results": [
    {
      "_apiVersion": "1.0",
      "_createdAt": 1513007727220,
      "_createdBy": "todoClient",
      "_history": [],
      "_id": "d353bfe1-7ef2-42ca-baf1-77cff1de13e8",
      "_version": 1,
      "completed": false,
      "name": "Build a RESTful API with DADI API"
    }
  ]
}
```

You can get a list of the existing to-do items with `GET http://localhost:8081/1.0/todos/items`, or query a particular item by ID like `GET http://localhost:8081/1.0/todos/items/d353bfe1-7ef2-42ca-baf1-77cff1de13e8`.

Let's now create a user and assign our new to-do item to them. Similarly to the above, we fire a `POST` request to `1.0/todos/users`.

```
POST http://localhost:8081/1.0/todos/users

Accept: application/json
Authorization: Bearer 2c483e66-74c9-4535-ac7b-26b4ca53ef53
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john.doe@somedomain.tech"
}
```

```json
{
  "results": [
    {
      "_apiVersion": "1.0",
      "_createdAt": 1513008476627,
      "_createdBy": "todoClient",
      "_history": [],
      "_id": "2fe27c95-dbff-4945-80a8-52fc188f1038",
      "_version": 1,
      "email": "john.doe@somedomain.tech",
      "name": "John Doe"
    }
  ]
}
```

Documents are referenced by their unique ID, which means that we must take the ID of the user we've just created and add it to the `assignee` property of the to-do item to be assigned.

To modify an existing document, we fire a `PUT` request to the collection endpoint including in the body the fields to be updated along with their new values.

```
PUT http://localhost:8081/1.0/todos/items/d353bfe1-7ef2-42ca-baf1-77cff1de13e8

Accept: application/json
Authorization: Bearer 2c483e66-74c9-4535-ac7b-26b4ca53ef53
Content-Type: application/json

{
  "assignee": "2fe27c95-dbff-4945-80a8-52fc188f1038"
}
```

We should get back something like:

```json
{
  "results": [
    {
      "_apiVersion": "1.0",
      "_composed": {
          "assignee": "2fe27c95-dbff-4945-80a8-52fc188f1038"
      },
      "_createdAt": 1513007727220,
      "_createdBy": "todoClient",
      "_history": [
        "d353bfe1-7ef2-42ca-baf1-77cff1de13e8"
      ],
      "_id": "d353bfe1-7ef2-42ca-baf1-77cff1de13e8",
      "_lastModifiedAt": 1513008678684,
      "_lastModifiedBy": "todoClient",
      "_version": 2,
      "assignee": {
        "_apiVersion": "1.0",
        "_createdAt": 1513008476627,
        "_createdBy": "todoClient",
        "_history": [],
        "_id": "2fe27c95-dbff-4945-80a8-52fc188f1038",
        "_version": 1,
        "email": "john.doe@somedomain.tech",
        "name": "John Doe"
      },
      "completed": false,
      "name": "Build a RESTful API with DADI API"
    }
  ],
  "metadata": {
    "page": 1,
    "offset": 0,
    "totalCount": 1,
    "totalPages": 1
  }
}
```

If you now do a `GET http://localhost:8081/1.0/todos/items/d353bfe1-7ef2-42ca-baf1-77cff1de13e8` you'll notice that the `assignee` field will show the ID of the assigned user and not the resolved reference. This is the default behavior that can be changed by adding `?compose=true` to the request URL or by setting the `compose` key to `true` in the `settings` block of the collection schema.

## Listing to-dos

Let's add a few more to-dos. We can do a bulk insert by suppling an array in the `POST` request, instead of a single object.

```
POST http://localhost:8081/1.0/todos/items

Accept: application/json
Authorization: Bearer 2c483e66-74c9-4535-ac7b-26b4ca53ef53
Content-Type: application/json

[
  {
    "name": "Learn about API authentication",
    "completed": true
  },
  {
    "name": "Learn about API collections",
    "completed": true
  },
  {
    "name": "Insert some data",
    "completed": true
  }
]
```

When querying a collection, you can specify a variety of filters to narrow down the result set to be exactly what you want. For example, if you wanted to get just the completed to-do items, you could do:

```
GET http://localhost:8081/1.0/todos/items?filter={"completed":false}
```

And to get all to-do items that contain the term `API` in the name:

```
GET http://localhost:8081/1.0/todos/items?filter={"name":{"$regex":"API"}}
```

## Wrapping up

We saw how to create a RESTful API and start inserting and querying data in just a few steps. Our [documentation pages](http://docs.dadi.tech/api) should be your next stop if you want to dive deeper into the product.

You should now be ready to take the `Build a RESTful API with DADI API` to-do we created and mark it as complete. I'll leave that one with you, I'm sure you'll nail it.