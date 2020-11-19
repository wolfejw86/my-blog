---
title: "Stream A Large Amount of Data From MongoDB Into A CSV"
tags: ["mongodb"]
published: true
date: "2019-10-12"
layout: layouts/post.njk
---

I recently needed to solve a problem with very large files using NodeJS streams. I learned several techniques, one of which in particular could really come in handy when you need to export a large amount of data from mongo and could run into data size limitations. In comes NodeJS streams and the MongoDB NodeJS driver to the rescue.

Let's start with the problem:

- I had a database with a large number of user documents that I wanted to parse into a CSV. How could I export them with a single query into a file without running into aggregate / query data size limits as well as limits on how much JSON I can send via a network request?

### Here Come NodeJS Streams

A brief high level overview of streams:

- They let you read from and write to data sources (usually files but can be other things as well)

- When reading from a stream, you get to "act" on a "chunk" of the data, and can even take action on it while reading from it. The real magic happens when you read from a stream and then transform it chunk by chunk while writing it to a writeable stream (usually an output file of some sort, in our case for the problem stated above a .csv file)

- The MongoDB driver in node has a built in streaming mechanism, where you can read the result of your query as a stream (and even optionally pass it a transform function to transform your data exactly how you want. This is how we will solve our problem above.)

### Solving The Problem

First we need a quick database setup / running locally. [I usually opt for a replica set](https://jaywolfe.dev/running-a-local-replica-set/). After you have a local setup, we need a .js file and an npm package. Since I'm demonstrating solving this problem for user data, I'm not going to use real data, but rather bulk create fake users. For that, I'll use `faker`. I'll also need `mongodb`, the node mongodb driver. Per usual, lets create a database connection.

```js
async function basicStreamExample() {
  const client = await new MongoClient('mongodb://localhost:27017/csv_test', {
    useNewUrlParser: true,
    useUnifiedTopology: true,
  }).connect();

  const db = client.db('csv_test');
```

Notice how I specified the db I want to use.

Let's create some fake users in memory. If you really want to test out the power of streams, change the number of fake users you create to an insanely large number, and watch how your Node process will still handle creating a CSV file out of your data with ease! For this demo, I'm just going to create 5000. We need a fake user creation function and a quick way to create them all.

```js
/**
 * creates a random user object with fake data
 */
function createRandomUser() {
  return {
    name: `${faker.name.firstName()} ${faker.name.lastName()}`,
    email: faker.internet.email(),
  }
}
```

We also need to actually populate the users so we can easily insert them.

```js
// create the fake users in memory first
const fakeUsers = new Array(5000).fill(0).map(createRandomUser)

// insert them into our database
await db.collection("users").insertMany(fakeUsers)
```

Now we have 5000 users in the db! Very important note, a stream is not really required for this few of users. My use case for the actual problem was a much higher number! The technique is still applicable though!

MongoDB's driver has a built-in technique to read the result of a query as a stream, and provide a transformation function! What does this mean!? It means we can transform documents into lines in a CSV file seamlessly! Check it out:

```js
/**
 * turn each document into a comma separated value followed by an end-of-line character
 */
function streamTransformer(doc) {
  return `${Object.values(doc).join(", ")}${os.EOL}`
}

// find all users, read as a stream, use our transformation function on each document "chunk"
const stream = db
  .collection("users")
  .find()
  .stream({ transform: streamTransformer })
```

Dissecting the above, we have a simple function that turns our user document (really could be any document) into a comma separated value followed by a newline. (Using NodeJS' `os` module to get the end-of-line character handles some edge cases for you which is nice.)

We then make our query, specify we want the result as a stream, and tell it to transform each chunk of the stream using our function above.

Now we still need to put the transformed data somewhere that it's going to be useful. The whole point of this is to create a big csv file right? Well here we go:

```js
const stream = db
  .collection("users")
  .find()
  .stream({ transform: streamTransformer })

const output = fs.createWriteStream("./users.csv")
stream.pipe(output)
```

We take our query stream, and `pipe` it into a writeable stream. What does this mean exactly? On a high level, each document of the query result will be processed one at a time with the transformation function, and then written to the write stream CSV file automatically. This is it! It's that simple! We do have one last case to handle, and that's "waiting" for the stream to be done so we know if it was succesful or not and when we can send the file. Good thing nodejs streams have a built-in `end` event we can "listen" to. For things like this, I like using Promises to make it a bit less nested with callbacks. I personally find it easier to read that way. Either will work! Our final section of the code can look something like this:

```js
const stream = db
  .collection("users")
  .find()
  .stream({ transform: streamTransformer })

const output = fs.createWriteStream("./users.csv")
const streamFinished = new Promise(resolve => stream.on("end", resolve))

stream.pipe(output)

await streamFinished

// go about your business as usual
```

There you have it! An easy way to write very large files. You could write them to a temporary folder and then send them to the user that way or even pipe them straight into the `Response` object if you're inside of a REST endpoint handler. I hope you enjoyed this small example and find it useful. [You can check out the full code on my Github here.](https://github.com/wolfejw86/blog-examples/blob/master/mongo-csv-example.js)
