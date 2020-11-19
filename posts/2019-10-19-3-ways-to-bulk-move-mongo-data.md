---
title: "4 Ways to Move Your MongoDB Data in Bulk - Part 1 - mongodump/mongorestore"
tags: ["mongodb"]
published: true
date: "2019-10-19"
layout: layouts/post.njk
---

Ever tried to migrate data from one database to another with MongoDB? How about bulk importing a lot of data to start building an app around? It can be a bit tricky no matter what kind of database you're using, however MongoDB has a lot of strategies you can use to handle this situation.

### Setup For Success (if you already know how to run 2 local mongods then skip to the next section)

I'm going to show you four different ways to move data around in MongoDB. To test out these strategies locally, you can either:

1.  Setup 2 separate MongoDB deployment projects in MongoDB Atlas (this method is probably a lot quicker if you're not familiar with running mongodb locally)

2.  Run 2 local mongodb instances locally. I won't go into the super detailed setup process as this isn't the goal of this article, but I will outline a few steps at a high level to accomplish 2 localhost mongodb deployments.

- start up 2 separate mongod processes locally in 2 separate terminals and separate them by ports.

  - I like to make the data directories really easy to find, so I usually do:

    - `mkdir -p ~/Desktop/mongo1/data/db` and `mkdir -p ~/Desktop/mongo2/data/db`

  - Now I can startup 2 mongod processes with those directories pretty quickly:

    - `mongod --port 20000 --dbpath ~/Desktop/mongo1/data/db`

    - `mongod --port 20001 --dbpath ~/Desktop/mongo2/data/db`

- refer to [this article](https://jaywolfe.dev/running-a-local-replica-set/) to run a replica set locally with docker-compose. To start a second replica set with docker-compose, just copy the repo folder for the first one into a separate location. Everywhere you see port 20000 in the first one, change it to a separate port in the second one (such as 20001). After doing this, you will have 2 connection strings `mongodb://localhost:20000/db` and `mongodb://localhost:20001/db`

### Using mongodump and mongorestore

There are several ways to configure running mongodump and mongorestore, but the biggest zinger I've found is being able to complete the whole entire operation in a single line in the terminal! Fortunately for me, it also fits my most common use case: Backing up our development environment onto my local machine to troubleshoot some issue that QA found with a feature!

First things first, lets get some real data that we can play with (and use throughout this entire series!). One of the easiest ways is to leverage MongoDB Atlas' sample datasets. If you spin up a cluster there, you can load sample data through their GUI for some interesting data sets to play with. One that I've found particularly interesting is the shipwrecks dataset, so I'll be using that here.

Get your mongodb connection string setup using a database user you create (the GUI can help you here too if you like) and double check that you can connect to it. I usually use:
```bash
mongo mongodb+srv://mflix-93ext.mongodb.net/sample_geospatial --username mongo_admin -p <password>
```
to confirm if it's working or not.

When the purpose of transferring data is for local debugging, you probably don't have SSL and auth setup locally which makes the command simpler to craft (although if you do, no worries, just check out the documentation to figure out the specific flags you'll need to set).

Now for the one liner to transfer all that data in one quick swoop!

```bash
mongodump --uri mongodb+srv://mongo_admin:<password>@your-host.net/sample_geospatial --archive | mongorestore --archive --uri mongodb://localhost:<desitination port>/sample_geospatial
```

Boom! Done!

Let's unpack it a little, because there is some nuance. The `|` operator pipes the output of mongodump straight into a mongorestore command, which is pretty cool! I'm no pipe pro, but the way I understand it is that it leverages stdout to not have to create permanent files on your machine. It reminds me of how NodeJS streams work at least in practice. `--archive` tells it to format as a `tar` file which probably aids in the piping effort. One little gotcha is that depending on which version of mongodb you're using, there can be weird errors. One I've gotten was when trying to rename the database on the mongorestore side of the command. It yelled at me that it was deprecated to specify a different db name, but it still worked! All in all this version of a dump makes the most sense if you're okay with keeping the database name the same.

Now that you have sample data locally, lets craft another transfer between separate dbs on our localmachine. If you followed the setup I detailed above, this should do the trick:

```bash
mongodump --uri mongodb://localhost:<port1>/sample_geospatial --archive | mongorestore --archive --uri mongodb://localhost:<destination port>/sample_geospatial
```

Nice! Now we can easily move data around without cluttering up our file system by storing mongodumps all over the place!

That's really all for this one folks! It's the fastest way I know of to get your data copied somewhere else in a quick manner. Maybe this one is obvious to everyone else, but I personally have referenced the mongodump/restore documentation pages thousands of times and never realized this little trick was a possibility. I hope it helps someone else out too! Stay tuned for part 2 coming up, where we'll talk about import/export techniques!

<!--
### Using mongoexport and mongoimport

I'm starting with import and export because it's the fastest way to get some initial data into your dbs, which will facilitate the rest of this article.

### Making Queries and Outputting to Files with mongo shell

### Writing to Large Files with Stream Data in NodeJS -->
