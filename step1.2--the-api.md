# Step 2 &mdash; The API Layer

There is now a functional database with a live model. The next thing to do it to write an API to access that model. But first, some background.

## What's an API?

The idea behind APIs is simple &mdash; **APIs give you an application's data**. Period. *What* you want to do with the data is up to you. You can display it on a web page; you can show parts of it on a mobile app; you can use bits from it to drive other pieces of your application, or *another* application &mdash; whatever you need to do. It's just plain, raw data. This even makes it possible to have different *views* of the data for different kinds of applications. For example, the [Weather API for The Weather Channel](https://www.wunderground.com/weather/api) gives you weather data. If you visit [weather.com](http://www.weather.com) on the desktop, you'll see that data presented differently than if you were to visit it on your mobile device. Same data, different representations. APIs make that possible. If you were only writing a weather *application*, that's *all* you'd be writing. It'd be a closed system, and no one else could use it in their *own* applications. By writing APIs, you let other people integrate your data &mdash; and maybe make money by allowing them that privilege.

### API example

So much for philosophy; let's see an example. Let's say you're writing an application to keep cooking recipes. Your application allows people to create their own recipes as well as find recipes that other people have created. You also want them to be able to update their own recipes, and leave comments or reviews for others. So recipes are your data, and creating, retrieving, editing, perhaps even deleting them, are the operations on that data. And if you think about it, most of what *every* application ever does are these four operations on some kind of data.

### Characteristics of RESTful APIs

At their most fundamental level, RESTful APIs define a convention for performing these operations on data. In a RESTful API implementation, each individual type of data is called a *resource*, and each operation that can be done on it &mdash; create, read, update and delete (or CRUD) &mdash; is indicated by means of an HTTP verb. HTTP *already* defines actions for each of those operations; REST just puts them in context of data instead of in context of a web page.

RESTful APIs have the following characteristics:

  * **They use HTTP methods** &mdash; `GET`, `POST`, `PATCH`, `DELETE`, etc. &mdash; to tell the application what to do. `GET` retrieves data; `POST` creates data; `PUT` and `PATCH` update data; and `DELETE` deletes data.

  * **They work on _resources_**. A *resource* is any data that your application wants to expose. In your application, a resource is a `Recipe` or an `Ingredient`. How *much* of that resource you want to expose depends on how you want other applications to be able to use it.

  * **They usually have a version identifier in the main part of the URI**. A common pattern is to start APIs with `/api/<version>`, so, for example, version 1 of your Recipes API will have its URIs starting with `/api/v1`.

  * **They identify resources with URIs**. For example, the `Recipe` resource might be identified by `/api/v1/recipes`. Resources are specified in plural &mdash; <code>/api/v1/<b>recipes</b></code> not <code>/api/v1/<b>recipe</b></code>.

  * **They allow resources to be "subordinate" to each other**. For example, it doesn't make sense to have an `Ingredient` unconnected to a `Recipe`, so you'll define `Ingredient` as a "component" of `Recipe`. This is achieved by creating URIs with multiple levels like so: `/api/v1/recipes/<recipe-id>/ingredients`. You can have as many levels of nesting as you want, but it's usually a good idea to keep it at two. It becomes cumbersome rather quickly to manage multiply nested resources.
 
  * **They package additional information to each action in query parameters**. For example, if you want to get only vegetarian Italian recipes, you could `GET` `/api/v1/recipes?type=vegetarian&cuisine=italian`. This keeps the call clean in that the API endpoint always stays `/api/v1/recipes`. Its *behaviour* can still be customized depending on the parameters passed to it.

### Interacting with the `Recipes` API

With all that in mind, let's see how a client might interact with your Recipes APIs for working with their recipe collection. I am using an client called [Postman](https://www.getpostman.com), but you can as well use [`curl`](https://curl.haxx.se).

The following table shows the connection between the URL a client uses for the `Recipe` resource, and the corresponding action that must happen:

Method|URL                   |Action                                |
------|----------------------|---------------------------------------
GET   |`/api/v1/recipes`     |Get a list of all recipes             |
POST  |`/api/v1/recipes`     |Add a new recipe to the list          |
GET   |`/api/v1/recipes/<id>`|Get details about a particular recipe |
PATCH |`/api/v1/recipes/<id>`|Update data about a particular recipe |
DELETE|`/api/v1/recipes/<id>`|Delete the recipe                     |

#### Get a list of all recipes

**Request** (Postman):

```bash
GET /api/v1/recipes HTTP/1.1
Host: 10.250.6.179:5000
Cache-Control: no-cache
Postman-Token: cb5bb6c4-9f54-ad52-e160-2ac481d1b428
```

**Request** (`curl`):

```bash
curl http://10.250.6.179:5000/api/v1/recipes
```

**Response** (JSON)

```json
{
  "request": {
    "uri": "/api/v1/recipes",
  },
  "meta": {
    "count": 615,
    "pages": 62,
    "next_page": "/api/v1/recipes&offset=10",
    "limit": 10,
    "offset": 0,
    "items_per_page": 10,
    "page": 1
  },
  "recipes": [
    {
      "uri": "/api/v1/recipes/1660539",
      "name": "Macaroni and cheese",
      "cuisine": "Italian",
      "num_reviews": 5,
      "ingredients": "/api/v1/recipes/1660539/ingredients"
    },
    {
      "uri": "/api/v1/recipes/1660540",
      "name": "Tomato soup",
      "cuisine": "American",
      "num_reviews": 2,
      "ingredients": "/api/v1/recipes/1660540/ingredients"
    },
    ...
  ]
}
```

Notice how subordinate resources (`Ingredient` in this example) are returned as another URL, which a client of the API can then use in the *very* same fashion as the main resource (`Recipe`).

Now *anyone* can use this data to build their own applications, like, for example, writing a grocery list app, or a website for clipping favourite recipes, or... who knows what else? And they don't even have to do in the language you used to write the API in.

APIs are the bedrock of a robust application. To find out more about how to build them, check out  [this excellent ebook](https://pages.apigee.com/web-api-design-register.html).


## The Books API

In a manner similar to that outlined above, our `Books` API looks like this:

Method|URL                 |Action                              |
------|--------------------|-------------------------------------
GET   |`/api/v1/books`     |Get a list of all books             |
POST  |`/api/v1/books`     |Add a new book to the list          |
GET   |`/api/v1/books/<id>`|Get details about a particular book |
PATCH |`/api/v1/books/<id>`|Update data about a particular book |
DELETE|`/api/v1/books/<id>`|Delete a particular book            |

To serve the API, we need Express, so install it for your application.

```bash
npm install express --save
```

and let's see how we can serve this basic API. Right now, let's not bother about code organization, or how to test this API &mdash; we'll come to those soon enough.

### Getting a list of books

Getting a list of books is one of the easiest of the five operations. There are no validations to make, no special cases, nothing at all but straight retrieval from the database. Here's the updated `main.js` from last time:

```js
require("dotenv").config();

const express = require("express");
const models = require("./models");

const app = express();

app.get("/books", async (request, response) => {
    var books = await models.Book.findAll();

    response.send({ books: books });
});
```

To use Express, import the Express module and then instantiate it. The instance variable is usually named `app`, but of course you can call it anything you like.

The function `app.get()` responds to a `GET` request, and it does so for the URL you define &mdash; in this case it is `/api/v1/books`. *What* gets done when someone visits that URL is defined by the second parameter to `app.get()`, the function that you write. In this case, we want to return a list of books from the database.

Now database calls are blocking calls, so you have to `await` any database operation. And since `await` can only be used inside functions that are explicitly marked asynchronous, this "handler" function has to be prefixed with the `async` keyword.

### Getting a particular book

Getting a single book is quite as easy as getting all of them &mdash; no big surprises there. Find the book given its id. If it's found, return it, otherwise return an error.

```js
app.get("/books/:id", async (request, response) => {
    var book = await models.Book.findById(parseInt(request.params.id));

    book ? response.send(book) : response.status(404).send("Not found");
});
```

### Creating a new book

Now we're getting into the more involved stuff. Not necessarily complicated, but definitely not the two line code as in the functions.

To create a new book, we have to accept a `POST` request. The URI is `/api/v1/books` &mdash; same as `GET`, but the _action_ is different. We also need to validate that the body of the `POST` request contains the title of the new book we need to add. Once that's validated, we can use the request to create a new `Book` object and save it to the database.

However, to extract the pieces of information we need from the body of the request, we need another NodeJS module called `body-parser`:

```bash
$ npm install body-parser --save
```

In `main.js`, import `body-parser` it and use it _before_ any of the handlers:

```js
require("dotenv").config();

const express = require("express");
const bodyParser = require("body-parser");
const models = require("./models");

const app = express();

app.use(bodyParser.json()); // for parsing JSON in the body of the request
app.use(bodyParser.urlencoded({ extended: true }));

app.get(...);
app.post(...); 

// ...

```
`body-parser` makes it easy to refer to the items in the request body &mdash; they can simply be referred to as `request.body.<param-name>`, like below:

```js
app.post("/books", async (request, response) => {
    if (!request.body.title)
    {
        response.status(400).send("Title is required.");

        return;
    }

    var newBook = new models.Book(request.body);
    newBook.save();

    response.send(newBook);
});
```

### Updating a particular book

Updating a book requires first looking it up, then changing the attributes specified in the body of the request, and finally saving it back to the database.

```js
app.put("/books/:id", async (request, response) => {
    // first, find the book, ...
    var book = await models.Book.findById(parseInt(request.params.id));
    if (!book)
    {
        response.status(400).send(`No book with id ${request.params.id} was found.`);

        return;
    }

    // ..., then set the attributes to change, ...
    if (request.body.title)
    {
        book.title = request.body.title;
    }

    // ..., and finally save the book back to the database, now with updated
    // attributes.
    await book.save();

    response.send(book);
});
```

### Deleting a particular book



In `main.js`, import Express

install express
installing nodemon

get /books
get /books/:id
post /books (json payload)
  body parser
  validations:
    show validations in request
    show validations in middleware

put /books/:id (json)
delete /books/:id

- ensure json payload (middleware)
- validate json payload for post
### write the api


### test the api with curl

### test the api with postman

### switch to web application
