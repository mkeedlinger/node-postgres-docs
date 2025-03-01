---
title: Suggested Project Structure
slug: /guides/project-structure
---

Whenever I am writing a project & using node-postgres I like to create a file within it and make all interactions with the database go through this file. This serves a few purposes:

- Allows my project to adjust to any changes to the node-postgres API without having to trace down all the places I directly use node-postgres in my application.
- Allows me to have a single place to put logging and diagnostics around my database.
- Allows me to make custom extensions to my database access code & share it throughout the project.
- Allows a single place to bootstrap & configure the database.

## example

_note: I am using callbacks in this example to introduce as few concepts as possible at a time, but the same is doable with promises or async/await_

The location doesn't really matter - I've found it usually ends up being somewhat app specific and in line with whatever folder structure conventions you're using. For this example I'll use an express app structured like so:

```
- app.js
- index.js
- routes/
  - index.js
  - photos.js
  - user.js
- db/
  - index.js <--- this is where I put data access code
```

Typically I'll start out my `db/index.js` file like so:

```js
const { Pool } = require('pg')

const pool = new Pool()

module.exports = {
  query: (text, params, callback) => {
    return pool.query(text, params, callback)
  },
}
```

That's it. But now everywhere else in my application instead of requiring `pg` directly, I'll require this file. Here's an example of a route within `routes/user.js`:

```js
// notice here I'm requiring my database adapter file
// and not requiring node-postgres directly
const db = require('../db')

app.get('/:id', (req, res, next) => {
  db.query('SELECT * FROM users WHERE id = $1', [id], (err, res) => {
    if (err) {
      return next(err)
    }
    res.send(res.rows[0])
  })
})

// ... many other routes in this file
```

Imagine we have lots of routes scattered throughout many files under our `routes/` directory. We now want to go back and log every single query that's executed, how long it took, and the number of rows it returned. If we had required node-postgres directly in every route file we'd have to go edit every single route - that would take forever & be really error prone! But thankfully we put our data access into `db/index.js`. Let's go add some logging:

```js
const { Pool } = require('pg')

const pool = new Pool()

module.exports = {
  query: (text, params, callback) => {
    const start = Date.now()
    return pool.query(text, params, (err, res) => {
      const duration = Date.now() - start
      console.log('executed query', { text, duration, rows: res.rowCount })
      callback(err, res)
    })
  },
}
```

That was pretty quick! And now all of our queries everywhere in our application are being logged.

<summary>
_note: I didn't log the query parameters.  Depending on your application you might be storing encrypted passwords or other sensitive information in your database.  If you log your query parameters you might accidentally log sensitive information.  Every app is different though so do what suits you best!
</summary>

Now what if we need to check out a client from the pool to run sever queries in a row in a transaction? We can add another method to our `db/index.js` file when we need to do this:

```js
const { Pool } = require('pg')

const pool = new Pool()

module.exports = {
  query: (text, params, callback) {
    const start = Date.now()
    return pool.query(text, params, (err, res) => {
      const duration = Date.now() - start
      console.log('executed query', { text, duration, rows: res.rowCount })
      callback(err, res)
    })
  },
  getClient: (callback) {
    pool.connect((err, client, done) => {
      callback(err, client, done)
    })
  }
}
```

Okay. Great - the simplesting thing that could possibly work. It seems like one of our routes that checks out a client to run a transaction is forgetting to call `done` in some situation! Oh no! We are leaking a client & have hundreds of these routes to go audit. Good thing we have all our client access going through this single file. Lets add some deeper diagnostic information here to help us track down where the client leak is happening.

```js
const { Pool } = require('pg')

const pool = new Pool()

module.exports = {
  query: (text, params, callback) {
    const start = Date.now()
    return pool.query(text, params, (err, res) => {
      const duration = Date.now() - start
      console.log('executed query', { text, duration, rows: res.rowCount })
      callback(err, res)
    })
  },
  getClient: (callback) {
    pool.connect((err, client, done) => {
      const query = client.query.bind(client)

      // monkey patch the query method to keep track of the last query executed
      client.query = () => {
        client.lastQuery = arguments
        client.query.apply(client, arguments)
      }

      // set a timeout of 5 seconds, after which we will log this client's last query
      const timeout = setTimeout(() => {
        console.error('A client has been checked out for more than 5 seconds!')
        console.error(`The last executed query on this client was: ${client.lastQuery}`)
      }, 5000)

      const release = (err) => {
        // call the actual 'done' method, returning this client to the pool
        done(err)

        // clear our timeout
        clearTimeout(timeout)

        // set the query method back to its old un-monkey-patched version
        client.query = query
      }

      callback(err, client, release)
    })
  }
}
```

That should hopefully give us enough diagnostic information to track down any leaks.
