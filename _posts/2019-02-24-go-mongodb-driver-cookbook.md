---
layout: post
title: Using the official MongoDB Go driver
slug: go-mongodb-driver-cookbook
date: 2019-02-24T10:56:41.000Z
img: mongo-gopher.png
tags: [Golang, MongoDB]
comments: true
excerpt: A cookbook on how use the official mongo (mongodb) driver for Go (golang).
---

I recently had to maintain an application that interfaces with a MongoDB database. This application was making use of the [mgo](https://github.com/go-mgo/mgo) library, which is now unmaintained, so I decided to start migrating the codebase to the new official mongo driver. 

Given it is fairly recent (the [first release](https://github.com/mongodb/mongo-go-driver/releases/tag/v0.0.1) is from Feb/2018), there aren't many tutorials around and although the [godocs](https://godoc.org/go.mongodb.org/mongo-driver/mongo) are very well written, I found I had to go through the source code to understand how to use it, especially when it comes to the BulkWrite API. So I thought it was worth writing a cookbook style post to help others (and myself in the future) get started with the most used operations.

# Installing

At the time of writing, the latest version of the driver is `1.0.0`. Let's setup [go modules](https://github.com/golang/go/wiki/Modules) on our project and install the [driver](https://github.com/mongodb/mongo-go-driver) using the following commands:

    $ go mod init github.com/victorkohl/mongo-driver-cookbook
    $ go get -u go.mongodb.org/mongo-driver/mongo@v1.0.0
    

# Connecting and disconnecting

There are two ways you can connect to the mongo database, you can either create a client and then connect to the database or use `mongo.Connect` function which will do both operations at once.

## Creating the client and then connecting

    // create a new context
    ctx := context.Background()
    
    // create a mongo client
    client, err := mongo.NewClient(
    	options.Client().ApplyURI("mongodb://localhost:6548/"),
    )
    if err != nil {
    	log.Fatal(err)
    }
    
    // connect to mongo
    if err := client.Connect(ctx); err != nil {
    	log.Fatal(err)
    }
    
    // disconnects from mongo
    defer client.Disconnect(ctx)
    

## Creating the client and connecting in a single step

    // create a new context
    ctx := context.Background()
    
    // create a mongo client
    client, err := mongo.Connect(
    	ctx,
    	options.Client().ApplyURI("mongodb://localhost:6548/"),
    )
    if err != nil {
    	log.Fatal(err)
    }
    
    // disconnects from mongo
    defer client.Disconnect(ctx)
    

## Selecting the database and collection

This is a straightforward task, just use `client.Database()` and `db.Collection()` functions:

    db := client.Database("blog")
    col := db.Collection("posts")
    

# CRUD operations

Now for the fun part, let's start interacting with our MongoDB database and do some [CRUD (Create, Read, Update, and Delete)](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations.

We will be using the following struct to decode the documents and `bson.M` type to insert/update/delete. You can, however, also use the struct for mutating your data. I wrote a section just to explain each serializing option, which can be found at the end of this post.

    type Post struct {
        ID        primitive.ObjectID `bson:"_id"`
        Title     string             `bson:"title"`
        Body      string             `bson:"body"`
        Tags      []string           `bson:"tags"`
        Comments  uint64             `bson:"comments"`
        CreatedAt time.Time          `bson:"created_at"`
        UpdatedAt time.Time          `bson:"updated_at"`
    }
    

## Insert one

To insert a document into a collection, use `collection.InsertOne()`:

    // create a new context with a 10 second timeout
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    // insert one document
    res, err := col.InsertOne(ctx, bson.M{
    	"title": "Go mongodb driver cookbook",
    	"tags":  []string{"golang", "mongodb"},
    	"body": `this is a long post
    that goes on and on
    and have many lines`,
    	"comments":   1,
    	"created_at": time.Now(),
    })
    if err != nil {
    	log.Fatal(err)
    }
    fmt.Printf(
    	"new post created with id: %s",
    	res.InsertedID.(primitive.ObjectID).Hex(),
    )
    // => new post created with id: 5c71caf32a346553363177ce
    

## Insert many

To insert many documents into a collection, use `collection.InsertMany()`:

    res, err := col.InsertMany(ctx, []interface{}{
    	bson.M{
    		"title":      "Post one",
    		"tags":       []string{"golang"},
    		"body":       "post one body",
    		"comments":   14,
    		"created_at": time.Date(2019, time.January, 10, 15, 30, 0, 0, time.UTC),
    	},
    	bson.M{
    		"title":      "Post two",
    		"tags":       []string{"nodejs"},
    		"body":       "post two body",
    		"comments":   2,
    		"created_at": time.Now(),
    	},
    })
    if err != nil {
    	log.Fatal(err)
    }
    fmt.Printf("inserted ids: %v\n", res.InsertedIDs)
    // => inserted ids: [ObjectID("5c71ce5c6e6d43eb6e2e93be") ObjectID("5c71ce5c6e6d43eb6e2e93bf")]
    

## Update one

To update a document, create the bson filters and update objects and pass to `collection.UpdateOne()`. Here we also create a new ObjectID from its hex string representation:

    // create ObjectID from string
    id, err := primitive.ObjectIDFromHex("5c71ce5c6e6d43eb6e2e93be")
    if err != nil {
    	log.Fatal(err)
    }
    
    // set filters and updates
    filter := bson.M{"_id": id}
    update := bson.M{"$set": bson.M{"title": "post 2 (two)"}}
    
    // update document
    res, err := col.UpdateOne(ctx, filter, update)
    if err != nil {
    	log.Fatal(err)
    }
    fmt.Printf("modified count: %d\n", res.ModifiedCount)
    // => modified count: 1
    

## Update many

To update multiple documents (equivalent to `{multi: true}`), use `collection.UpdateMany()`, which has the same signature as `UpdateOne`:

    // set filters and updates
    filter := bson.M{"tags": bson.M{"$elemMatch": bson.M{"$eq": "golang"}}}
    update := bson.M{"$set": bson.M{"comments": 0, "updated_at": time.Now()}}
    
    // update documents
    res, err := col.UpdateMany(ctx, filter, update)
    if err != nil {
    	log.Fatal(err)
    }
    fmt.Printf("modified count: %d\n", res.ModifiedCount)
    // => modified count: 17
    

## Upsert

For both `UpdateOne` and `UpdateMany` you can make it an upsert operation by passing the proper option to the function:

    res, err := col.UpdateOne(
    	ctx, filter, update, options.Update().SetUpsert(true)
    )
    
    res, err := col.UpdateMany(
    	ctx, filter, update, options.Update().SetUpsert(true)
    )
    

## Delete one

To delete a document, use `collection.DeleteOne()`:

    // create ObjectID from string
    id, err := primitive.ObjectIDFromHex("5c71ce5c6e6d43eb6e2e93be")
    if err != nil {
    	log.Fatal(err)
    }
    
    // delete document
    res, err := col.DeleteOne(ctx, bson.M{"_id": id})
    if err != nil {
    	log.Fatal(err)
    }
    fmt.Printf("deleted count: %d\n", res.DeletedCount)
    // => deleted count: 1
    

## Delete many

Finally, to delete multiple documents, use `collection.DeleteMany()`:

    // delete documents created older than 2 days
    filter := bson.M{"created_at": bson.M{
    	"$lt": time.Now().Add(-2 * 24 * time.Hour),
    }}
    
    // update documents
    res, err := col.DeleteMany(ctx, filter)
    if err != nil {
    	log.Fatal(err)
    }
    fmt.Printf("deleted count: %d\n", res.DeletedCount)
    // => deleted count: 7
    

## Find one

Now for the read operations the signature is a bit different. For methods that return a single item, a `SingleResult` instance is returned. Here we will be decoding the result into our `Post` struct defined in the beginning of this post.

    // filter posts tagged as golang
    filter := bson.M{"tags": bson.M{"$elemMatch": bson.M{"$eq": "golang"}}}
    
    // find one document
    var p Post
    if err := col.FindOne(ctx, filter).Decode(&p); err != nil {
    	log.Fatal(err)
    }
    fmt.Printf("post: %+v\n", p)
    

## Find many

To retrieve multiple documents, you'll use a cursor to iterate through the results and decode it into our struct again:

    // filter posts tagged as golang
    filter := bson.M{"tags": bson.M{"$elemMatch": bson.M{"$eq": "golang"}}}
    
    // find all documents
    cursor, err := col.Find(ctx, filter)
    if err != nil {
    	log.Fatal(err)
    }
    
    // iterate through all documents
    for cursor.Next(ctx) {
        var p Post
        // decode the document
        if err := cursor.Decode(&p); err != nil {
        	log.Fatal(err)
        }
        fmt.Printf("post: %+v\n", p)
    }
    
    // check if the cursor encountered any errors while iterating 
    if err := cursor.Err(); err != nil {
    	log.Fatal(err)
    }
    

# Bulk writes

Sometimes you want to do many operations like inserting, updating and deleting all at once. In this case MongoDB provides the very useful [bulk write API](https://docs.mongodb.com/manual/reference/method/db.collection.bulkWrite/), which takes an array of write operations and executes each of them. By default operations are executed in order. Here is an example:

    // list of inserts
    inserts := []bson.M{
    	{
    		"title":      "post five",
    		"tags":       []string{"postgresql"},
    		"created_at": time.Now(),
    	},
    	{
    		"title":      "post six",
    		"tags":       []string{"graphql"},
    		"created_at": time.Now(),
    	},
    }
    
    // list of updates
    updates := []struct {
    	filter  bson.M
    	updates bson.M
    }{
    	{
    		filter: bson.M{
    			"tags": bson.M{"$elemMatch": bson.M{"$eq": "golang"}},
    		},
    		updates: bson.M{"$set": bson.M{"updated_at": time.Now()}},
    	},
    }
    
    // list of deletes
    deletes := []bson.M{
    	{"_id": id1},
    	{"_id": id2},
    	{"_id": id3},
    }
    
    // create the slice of write models
    var writes []mongo.WriteModel
    
    // range over each list of operations and create the write model
    for _, ins := range inserts {
    	model := mongo.NewInsertOneModel().SetDocument(ins)
    	writes = append(writes, model)
    }
    for _, upd := range updates {
    	model := mongo.NewUpdateManyModel().
    		SetFilter(upd.filter).SetUpdate(upd.updates)
    	writes = append(writes, model)
    }
    for _, del := range deletes {
    	model := mongo.NewDeleteManyModel().SetFilter(del)
    	writes = append(writes, model)
    }
    
    // run bulk write
    res, err := col.BulkWrite(ctx, writes)
    if err != nil {
    	log.Fatal(err)
    }
    fmt.Printf(
    	"insert: %d, updated: %d, deleted: %d",
    	res.InsertedCount,
    	res.ModifiedCount,
    	res.DeletedCount,
    )
    // => insert: 2, updated: 10, deleted: 3
    

# Serializing/deserializing and BSON types

[Package bson](https://github.com/mongodb/mongo-go-driver/blob/master/bson/bson.go) is a library for reading, writing, and manipulating BSON. I think it deserves a special mention, as you'll be using it a lot and their types can be a bit confusing on their purpose.

### bson.M

As per [godocs](https://godoc.org/go.mongodb.org/mongo-driver/bson#M), `bson.M` is an unordered, concise representation of a BSON Document. It should generally be used to serialize BSON when the order of the elements of a BSON document do not matter.

    _, err := col.InsertOne(ctx, bson.M{
        "title":      "post",
        "tags":       []string{"mongodb"},
        "body":       `blog post`,
        "created_at": time.Now(),
    })
    

The above operation will create a document with unordered elements, for example:

    {
        "_id" : ObjectId("5c71f03ccfee587e4212ad8e"),
        "body" : "blog post",
        "created_at" : ISODate("2019-02-24T01:15:40.326Z"),
        "title" : "post",
        "tags" : [ 
            "mongodb"
        ]
    }
    

Notice the elements have different order than the ones provided in the `bson.M`.

### bson.D

`bson.D` represents a BSON Document. This type can be used to represent BSON in a concise and readable manner.

    _, err := col.InsertOne(ctx, bson.D{
        {"title", "post 2"},
        {"tags", []string{"mongodb"}},
        {"body", `blog post 2`},
        {"created_at", time.Now()},
    })
    

This operation will create a document with ordered elements, for example:

    {
        "_id" : ObjectId("5c71f03ccfee587e4212ad8f"),
        "title" : "post 2",
        "tags" : [ 
            "mongodb"
        ],
        "body" : "blog post 2",
        "created_at" : ISODate("2019-02-24T01:15:40.328Z")
    }
    

### Struct tags

You can also use `bson` struct tags on your struct to encode its values to BSON. Given our `Post` struct defined previously, let's create another document:

    _, err := col.InsertOne(ctx, &Post{
        ID:        primitive.NewObjectID(),
        Title:     "post",
        Tags:      []string{"mongodb"},
        Body:      `blog post`,
        CreatedAt: time.Now(),
    })
    

And we get the following document:

    {
        "_id" : ObjectId("5c71f03ccfee587e4212ad90"),
        "title" : "post",
        "body" : "blog post",
        "tags" : [ 
            "mongodb"
        ],
        "comments" : NumberLong(0),
        "created_at" : ISODate("2019-02-24T01:15:40.329Z"),
        "updated_at" : null
    }
    

As you can see, the driver will create a document with all fields defined in the struct, even the ones we didn't pass any values to. Keep in mind that you'll have to use the `primitive.NewObjectID()` function to create a new ObjectID. It will also preserve the order of the properties defined in the struct.

# Conclusion

Hopefully this post can help you get started quickly with the most common type of operations you might need when using the MongoDB driver with Go.

You can find the full source code in my [Github repository](https://github.com/victorkohl/mongo-driver-cookbook).

** Mongo Gopher Artwork, credits to [@ashleymcnamara](https://github.com/ashleymcnamara)*
