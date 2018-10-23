# Step 1 &mdash; The Data Layer

OK, so you have NodeJS installed and working, you have Express installed and working, you also have a working database. With these preliminaries out of the way, you can actually start writing the application. But hold on, let's take it in bite-sized pieces.

For this very first iteration, all we want to do is manage a list of books. Nothing more &mdash; no authors, no users, no reading lists, nothing. Just the four CRUD operations on a collection of `Book` objects. And all each `Book` has is just a `title`.

## The database

Well, that's simple, you say. If that's all we're doing, let's jump right in and create the `books` table in the database, and call it a day.

You're right, creating a `books` table is indeed what you want to do &mdash; except you also want to keep an eye on the future.

### Databases and ORMs

As you build your application, your database structure will change &mdash; that's just a given. You'll add more tables; you'll add or remove columns from tables; you'll define relationships between tables &mdash; it's a dynamic, evolving system. You need a tool to manage that evolution; doing it manually will become cumbersome, fast.

Besides, in the application you'll be using JavaScript objects for the data, but in the database, the data will be organized in rows and columns, *very* unlike an object. You'll need to manage the difference in representation of the data that the application sees versus that which the database sees. Let's see an example &mdash; this will also be a little glimpse of the future.

Say in your application, a `Book` object has a list of `Author`s indicating which authors wrote that book. The `Author` object, similarly, has a list of *`Book`s* that author wrote. The application might model it like this:

```js
// The Book model

const book = {
  title: "Head First Design Patterns",
  authors: [
    "Eric Freeman",
    "Elisabeth Robson"
  ]
};
```

```js
// The Author model

const author = {
  name: "Eric Freeman",
  books: [
    "Head First Design Patterns",
    "Head First JavaScript Programming",
    "Head First HTML and CSS"
  ]
};
```

In the database, however, this information is represented differently, in rows and columns:

The `authors` table:

| id | name            |
|---:|-----------------|
|5   |Eric Freeman     |
|8   |J. K. Rowling    |
|9   |Elisabeth Robson |
|10  |E. R. Braithwaite|

The `books` table:

| id | title                                   |
|---:|-----------------------------------------|
|3   |Head First Design Patterns               |
|6   |To Sir, With Love                        |
|8   |Harry Potter and the Prisoner of Azkaban |
|11  |Head First JavaScript Programming        |
|12  |Head First HTML and CSS                  |


The `authors_books` association table (defines the many-to-many *associations* between `books` and `authors`):

| book_id | author_id |
|--------:|----------:|
|3        |5          |
|3        |9          |
|6        |10         |
|8        |8          |
|11       |5          |
|12       |5          |

As you might gather from these two drastically different data representations, translating the application’s data model to the database’s data model is a tedious task. Automation is the best remedy for tedium, and for this situation, that remedy is called an Object Relational Mapper, or ORM. You tell the ORM how your data looks, how the relationships between the different objects in your application look, and the ORM will figure out how to map the application’s data to the database and back. 

But that’s not all an ORM is good for. Where an ORM really shines is in managing the incremental modifications the database goes through as it changes and evolves. That's called database migration, and we'll get to that in a bit.

The most popular ORM for Postgres for NodeJS is `Sequelize`, and that's what we'll be using here.

### The `Sequelize` ORM

The first step, as you know by now, is to install the `sequelize` package, and have it added to `package.json`:

```bash
$ npm install sequelize --save
```

In addition, you also have to install the libraries for Postgres, which `Sequelize` will use:

```bash
$ npm install pg pg-hstore --save
```

At this point, you can define your tables using `Sequelize` [from within your application](http://docs.sequelizejs.com/manual/installation/getting-started), but, again, with an eye to the future, let's not only *define* our tables using `Sequelize`, but also set up to *migrate* the database when the time comes.

To run migrations, we need the command-line interface to `Sequelize`, called `sequelize-cli` ([link](http://docs.sequelizejs.com/manual/tutorial/migrations.html)). So let's install that:

```bash
$ npm install sequelize-cli --save-dev
```

You don't need `sequelize-cli` on a running system, only on a development system, so the command above tells `npm` to install it only for development. You can see that after you run the command, `sequelize-cli` will have been added to the `devDependencies` section of `package.json` instead of to the `dependencies` section.

Then initialize the `Sequelize` migration and database management system by running `sequelize init`. The `sequelize` command is in your `node_modules` directory, so run it like this:

```bash
$ ./node_modules/.bin/sequelize init
```

After running this command, you'll find four new directories created for you:

```bash
config/
  config.json
migrations/
models/
seeders/
```

Each of these directories has a particular purpose. `migrations` holds the database migrations, `models` contains the model objects (objects representing the database tables), and `seeders` contains the initial "seed" data, if any. The `config` directory holds configuration information in the form of the `config.json` file:

```json
{
  "development": {
    "username": "root",
    "password": null,
    "database": "database_development",
    "host": "127.0.0.1",
    "dialect": "mysql",
  },
  "test": {
    "username": "root",
    "password": null,
    "database": "database_test",
    "host": "127.0.0.1",
    "dialect": "mysql"
  },
  "production": {
    "username": "root",
    "password": null,
    "database": "database_production",
    "host": "127.0.0.1",
    "dialect": "mysql"
  }
}
```
For now, these are all dummy, placeholder values. Since we're doing this only for development, get rid of all other keys but `development` and change those settings to the actual values for the database you've configured:

```json
{
  "development": {
    "username": "<your user name>",
    "password": "<your password>",
    "database": "<your database>",
    "host": "<your database host>",
    "dialect": "postgres",
    "ssl": true,
    "dialectOptions": {
      "ssl": true
    },
    "define": {
      "underscored": true
    }
  }
}
```

For remote, hosted servers, the `ssl` value should be present in *two* places as above and be `true` in both too (link to this issue). For local connections, you can omit both. The `underscored: true` value tells `Sequelize` to create tables with columns named in `snake_case` (with underscores) instead of in `camelCase`. You can leave that out too if you like your column names in `camelCase`. I happen to dislike them.

Now that the configuration is taken care of, let's define a new model &mdash; the `Book` model. In the terminal, run:

```bash
$ ./node_modules/.bin/sequelize model:generate --name Book --attributes "title:string"
```

The `model:generate` command tells `Sequelize` to generate a model called `Book` that has the `title` attribute, which is a `string`. And sure enough, after this command, there's a file called `book.js` in your `models` directory (and an `index.js`, which we'll talk about in a bit). There's *also* a new migration in the `migrations` directory, called `20181005183603-create-book.js` (your name will vary because the numeric prefix, as you have undoubtedly noticed, is the current date and time stamp).

Let's check these files out.

`book.js`, which I've tweaked a bit, looks like this:

```js
'use strict';

module.exports = (sequelize, DataTypes) => {
  const bookSchema = {
    title: DataTypes.STRING
  };

  const Book = sequelize.define('Book', bookSchema, { tableName: "books" });

  Book.associate = function(models) {
    // associations can be defined here
  };

  return Book;
};

```
I like to keep the schema definition separate from the `sequelize.define()` call, and I like to name my tables explicitly, so those are the changes I made. The `Book` model, I told `Sequelize`, corresponds to rows in the `books` table.

The migration script, `20181005183603-create-book.js` looks like this:

```js
'use strict';

module.exports = {
  up: (queryInterface, Sequelize) => {
    return queryInterface.createTable('books', {
      id: { type: Sequelize.INTEGER, allowNull: false, autoIncrement: true, primaryKey: true },
      title: { type: Sequelize.STRING, allowNull: false },
      created_at: { type: Sequelize.DATE, allowNull: false },
      updated_at: { type: Sequelize.DATE, allowNull: false }
    });
  },

  down: (queryInterface, Sequelize) => {
    return queryInterface.dropTable('books');
  }
};

```

All *that*'s doing is defining what happens when you `up`grade or `down`grade your database with this migration. Upgrading will create a new table called `books` (this must match the table name in the model) with the specified columns, and downgrading will do the opposite &mdash; drop that table. You'll also notice that there are three other columns in the table that you never specified in your `model:create` command &mdash; `id`, `created_at` and `updated_at`. These are columns that `model:create` adds by default. There are ways to override that default behaviour, but you almost always want to keep the `id` column, if not the `created_at` and `updated_at` columns.

And now, for the moment of truth. Run the migration, and behold the newly created table in your database without your having to write the SQL for doing it yourself.

```bash
$ ./node_modules/.bin/sequelize db:migrate
```

Here's the output:


```pre
Sequelize CLI [Node: 8.11.3, CLI: 4.1.1, ORM: 4.39.0]

Loaded configuration file "config\config.json".
Using environment "development".
sequelize deprecated String based operators are now deprecated. Please use Symbol based operators for better security, read more at http://docs.sequelizejs.com/manual/tutorial/querying.html#operators node_modules\sequelize\lib\sequelize.js:242:13
== 20181005183603-create-book: migrating =======
== 20181005183603-create-book: migrated (0.116s)
```
You can now verify that in your database, there indeed is a `books` table. You didn't have to write a line of SQL. Admittedly, this was more work than merely writing that `CREATE TABLE` SQL, but the real rewards of this system will appear when you have to change table structures and relationships manually. But even now, you can see that if you have to remove this table, you can merely say:

```bash
$ ./node_modules/.bin/sequelize db:migrate:undo
```

Each call to `db:migrate:undo` undoes the last run migration, no matter how many migrations you've run. Check it out now. Run this command, and see the `books` table disappear from your database (then run `db:migrate`, and it'll be added to the database again).

## Testing the database setup

Before wrapping up, let's check if this whole system actually does work. So create a `main.js` file in the application directory (or whatever name you specified in `package.json` as the value of the `main` key. You can change that at will). Here's the code to add a new book to the database:

```js
const models = require("./models");

var newBook = new models.Book({title: "Harry Potter and the Prisoner of Azkaban"});
newBook.save();
```

Yes, that's it. No opening connections to the database, no database calls, no SQL writing, nothing whatsoever to do with the database itself. Just instantiating a model object, setting some properties, and saving it, all in three lines of code. Run this tiny application (`node main.js`), and behind the scenes, `Sequelize` will connect to the database, generate the SQL, and execute that query:

```pre
sequelize deprecated String based operators are now deprecated. Please use Symbol based operators for better security, read more at http://docs.sequelizejs.com/manual/tutorial/querying.html#operators node_modules\sequelize\lib\sequelize.js:242:13
Executing (default): INSERT INTO "books" ("id","title","created_at","updated_at") VALUES (DEFAULT,'Harry Potter and the Prisoner of Azkaban','2018-10-08 18:09:00.874 +00:00','2018-10-08 18:09:00.874 +00:00') RETURNING *;
```

(If you're concerned about that warning above: *sequelize deprecated String based operators are now deprecated*, [here's an explanation and a solution](https://github.com/sequelize/sequelize/issues/8417)).

A couple of things to notice here. First, you didn't need to import the `Book` model. There's no line that says:

```js
const Book = require("./models/book");
```

You only imported the `models` directory. How did that work? Remember the `index.js` file that got generated as part of `sequelize init`? That's what pulled in all files that had a `.js` extension (in this case only `book.js`), exported their code, and made them available. You'll never have to import individual models &mdash; you can just import the `models` directory (which imports `index.js`), and all model objects will become available as members of the `models` object.

Secondly, notice that the `id`, `created_at` and `updated_at` fields that you never defined in the model are being automatically created and managed, thanks to the migration that `Sequelize` generated for you.

### Dynamic configuration

One final topic before moving on to writing the APIs. Currently, your database credentials are stored in the config file, `config.json`. Consequently, this information is checked into version control and becomes part of the source code. This is [not deemed best practice](https://12factor.net/config) &mdash; sensitive information like passwords and other kinds of credentials should be in environment variables. And each set of environment variables should be set up during deployment for that particular runtime environment. For example, on the production server, the environment variables hold production credentials; for the test server, they hold credentials for the test environment, both of which are different from each other.

We can make use of the customization that `Sequelize` allows to put database credentials into environment variables. First, we need to create a file named `.sequelizrc`. Despite its name, this is a JavaScript file (yes, it was baffling to me too). This file holds custom settings for `Sequelize`. We'll use this to specify a different configuration file:

```js
// .sequelizerc

const path = require("path");

module.exports = {
    config: path.resolve("config", "config.js")
};
```

`Sequelize` can load settings from JavaScript files as well as from JSON files, so the code above tells `Sequelize` to load its settings from `config.js` instead of from the (default) `config.json` file.

Then, we create `config.js` in the `config` directory and structure it just like `config.json` EXCEPT that the values of the variables come from the environment (and that the keys don't have quotes around them):

```js
module.exports = {
  development: {
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DATABASE,
    host: process.env.DB_HOST,
    dialect: "postgres",
    ssl: true,
    dialectOptions: {
      ssl: true
    },
    define: {
      underscored: true
    }
  }
```

Notice that the values of the credentials now come from the `process.env` variable that NodeJS makes available. We haven't defined those variables yet &mdash; that's the next step.

Next, create a file in the root of your workspace named `.env`, and add the environment variables there:

```pre
DB_USERNAME=frimcgmpagcllv
DB_PASSWORD=78f6adb33a0b968ff81442cba00275ca24880bec6afc37da027290e2be81adbc
DATABASE=d6ko6na6npsjn7
DB_HOST=ec2-50-17-194-186.compute-1.amazonaws.com
DB_DIALECT=postgres
```

Then, install the `dotenv` package, and include it as the very first thing in your main program (`main.js`):

```bash
$ npm install dotenv --save
```

```js
require("dotenv").config();

const models = require("./models");

var newBook = new models.Book({title: "Harry Potter and the Prisoner of Azkaban"});
newBook.save();
```

And finally, tweak `models/index.js` by replacing:

```js
const config = require(__dirname + '/../config/config.json')[env];
```

with:

```js
const config = require(__dirname + '/../config/config.js')[env];
```

Reminder: Do NOT commit `.env` to version control; it contains sensitive information and should be present only on the actual system that's running your application.

And now, we're finally ready for the APIs.
