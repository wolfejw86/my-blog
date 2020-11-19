---
title: "Setup Your First Change Stream"
tags: ["mongodb"]
published: true
date: "2019-09-10"
layout: layouts/post.njk
---

Alright, it's time to use that replica set we recently learned how to stand up! If you haven't set one up yet or want to learn more about how to run one locally then [take a look at my previous post here on standing up a local replica set](/running-a-local-replica-set/). If you want to skip straight to the repo, you can clone it here - https://github.com/wolfejw86/local-replica-set and just run `docker-compose up` and it should just work. Otherwise take a look below as I dive into the code for a bit of a deeper explanation.

```js
const crypto = require("crypto")
const MongoClient = require("mongodb").MongoClient

const url = "mongodb://localhost:27017"
const dbName = "myproject"
const client = new MongoClient(url, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
})

async function main() {
  await client.connect()
  console.log("db connected")

  const db = client.db(dbName)

  const collection = db.collection("changetest")
  const changeStream = collection.watch([], { fullDocument: "updateLookup" })

  const data = [
    { event: crypto.randomBytes(5).toString("hex") },
    { event: crypto.randomBytes(5).toString("hex") },
    { event: crypto.randomBytes(5).toString("hex") },
    { event: crypto.randomBytes(5).toString("hex") },
    { event: crypto.randomBytes(5).toString("hex") },
  ]

  let i = -1
  const interval = setInterval(async () => {
    await collection.insertOne(data[++i])

    if (i === data.length - 1) {
      clearInterval(interval)
      setTimeout(async () => {
        await collection.insertOne({ event: "last one" })
      }, 5000)
    }
  }, 1000)

  /* uncomment below during exercise */

  /*
  async function getNext() {
    const next = await changeStream.next();

    console.log("change stream fired", next.fullDocument.event);

    if (next) {
      return await getNext();
    }
  }

  await getNext();

  console.log("exercise complete, disconnecting");

  await client.close();
  */

  /* comment out the below when running the above chunk */
  changeStream.on("change", async data => {
    console.log("change stream fired", data.fullDocument.event)

    if (data.fullDocument.event === "last one") {
      console.log("exercise complete, disconnecting")
      changeStream.removeAllListeners()
      await client.close()
    }
  })
}

main()
```

Let's walk through this example really quickly. In this first section:

```js
const crypto = require("crypto")
const assert = require("assert")
const MongoClient = require("mongodb").MongoClient

const url = "mongodb://localhost:27017"
const dbName = "myproject"
const client = new MongoClient(url, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
})
```

I am simply requiring my dependencies. A lot of folks will use `mongoose` even for small examples like this. I like to step away from it every once in a while so that I remember that it's really a fancy wrapper around the native `mongodb` driver. When I remember this, I can always easily address inconsistencies with what I'm trying to do in mongo vs what the mongoose documentation is telling me. That being, said, this is simply setting up for a basic connection. Yes the uri is hardcoded, no you shouldn't do this in production ;) We just want to see something work that will _only_ work in a replica set and not on your simple single node localhost mongo!

```js
async function main() {
  await client.connect()
  console.log("connected to db")
```

Alright on to the main event! First off, now that the newest version of the mongodb driver for nodejs supports promises, we can just await the connection action to the db. Why am I using this `async function main` you ask? Because we have no top level await support in javascript. It would be nice but honestly, who cares?! Just put a single async function to initiate your "business" and get started. Just don't forget to call it to start your app!

```js
const db = client.db(dbName)

const collection = db.collection("changetest")
const changeStream = collection.watch([], { fullDocument: "updateLookup" })

const data = [
  { event: crypto.randomBytes(5).toString("hex") },
  { event: crypto.randomBytes(5).toString("hex") },
  { event: crypto.randomBytes(5).toString("hex") },
  { event: crypto.randomBytes(5).toString("hex") },
  { event: crypto.randomBytes(5).toString("hex") },
]
```

Now we have the next step in the setup process. Let's remember what we're here to do for a second.

- create a collection so that we can insert some fake data
- setup a change stream to watch the collection so we can see how they work
- insert some fake data and watch the change stream in action

The data array is literally just a list of objects with a single property (randomly generated hex string) so that we can see each unique value going into the database.

Alright now get ready for some first class hackery:

```js
let i = -1
const interval = setInterval(async () => {
  await collection.insertOne(data[++i])

  if (i === data.length - 1) {
    clearInterval(interval)
    setTimeout(async () => {
      await collection.insertOne({ event: "last one" })
    }, 5000)
  }
}, 1000)
```

Here I'm just setting up a basic interval to insert some fake data. On each pass of the interval (once per second) it will insert the current item in data into the array. There's a couple of ways to do this, but I don't find a way to use intervals too often so it's fun to play with them every once in a while. Once the incrementor `i` gets to the last item in the array, the interval just short circuits itself so that the program will stop while also firing one last insert.

Uncomment the below section of code if you're running this locally, or just follow along here:

```js
async function getNext() {
  const next = await changeStream.next()

  console.log("change stream fired", next.fullDocument.event)

  if (next) {
    return await getNext()
  }
}

await getNext()
```

Here we have a neverending promise await, which in theory is bad because it will never finish resolving. Notice how it even waits five seconds for the "last event" to fire, and then still the program doesn't exit. I mainly wanted to show that change stream events can be awaited in a programmatic way, however I haven't come up with a great pattern to manage this. If you know of a better use case for using the promise chain method of listening for change stream events please reach out and let me know!

I find it much easier to do this instead (don't forget to comment out the `getNext` function and where it gets called):

```js
changeStream.on("change", async data => {
  console.log("change stream fired", data.fullDocument.event)

  if (data.fullDocument.event === "last one") {
    console.log("exercise complete, disconnecting")
    changeStream.removeAllListeners()
    await client.close()
  }
})
```

Now we can just setup a listener on the change stream object and respond to it accordingly. It allows for a decent amount of abstraction as well, which can be helpful with loose coupling and testing your code. Notice how also it's really easy to disconnect when you want to, although you probably will not want to do this and instead have the change stream just running for the duration of your app.

That about does it! We have a basic change stream up and running, two ways to interact with it, and a little bit of nuance insight into what it's doing behind the scenes.
