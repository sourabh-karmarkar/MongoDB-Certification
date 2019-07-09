### Please refer the official website

https://university.mongodb.com/

### MongoDB-Certification

Repository for material and practice stuff for mongoDB certification

### Some important links used

- http://tuyenvuong.me/

### Connecting to univeristy.mongodb.com class atlas cluster from the mongo shell

- Please enter the following command in the command prompt. You can find the details of the command from the tutorial video

```
mongo "mongodb://cluster0-shard-00-00-jxeqq.mongodb.net:27017,cluster0-shard-00-01-jxeqq.mongodb.net:27017,cluster0-shard-00-02-jxeqq.mongodb.net:27017/100YWeatherSmall?replicaSet=Cluster0-shard-0" --authenticationDatabase admin --ssl --username m001-student --password m001-mongodb-basics
```

### Connecting to your mongodb atlas cluster from the mongo shell

- Please enter the following command in the command prompt. You can find the details of the command from your mongodb atlas account on clicking connect

```
mongo "mongodb+srv://cluster0-okti4.mongodb.net/test" --username database_username
```

### Loading JS files to your atlas cluster

- Connect to the mongo shell by going to the directory where the js file exists.
- Execute the following command after connecting to the mongo shell

```
load("file_name.js")
```

## MongoDB Basic shell commands (We are referring the video database from loadMovieDetailsDataset.js file, loadReviewsDataset.js file and the M001 Mongo Tutorial Cluster for these commands)

### View list of collections from the connected database

```
show collections
```

### Change the database

```
use database_name
```

### Show the list of all the documents in the specified collection

```
db.collection_name.find()
```

### Inserting a single document in the database

```
db.collection_name.insertOne({
    title: "Star Wars",
    year: 1982,
    imdb: "tt0084726"
})
```

### Inserting multiple documents in the database

- By default inserts multiple documents in ordered format, if it encounters error then it stops further insertion
    
```
db.collection_name.insertMany([{
        "_id": "tt0084726",
        "title": "Star Trek 2 : The Wrath of Khan",
        "year": 1982,
        "type": "movie"
    },
    {
        "_id": "tt0796366",
        "title": "Star Trek",
        "year": 2009,
        "type": "movie"
    },
    {
        "_id": "tt0084726",
        "title": "Star Trek 2 : The Wrath of Khan",
        "year": 1982,
        "type": "movie"
    }
], {
    "ordered": false
})
```

### Find documents based on some condition (scalar values)
    
```
db.movies.find({
    mpaaRating: "PG-13",
    year: 2009
})
```

### Find documents based on some condition (array fields)

- exact match

```
db.movies.find({
    cast: ["Jeff Bridges", "Tim Robbins"]
})
```

- array containing the specified value at any position.
```
db.movies.find({
    cast: "Jeff Bridges"
})
```

- array containing the specified value at a specified position.
```
db.movies.find({
    "cast.0": "Jeff Bridges"
})
```

### Cursors

- printing the cursor (by default prints 20 values, type "it" in the mongo shell to view the next 20 values and so on)
```
var myCursor = db.movies.find({
    cast: ["Jeff Bridges", "Tim Robbins"]
})

myCursor
```

- iterating the cursor (there are many ways to iterate, however below listed commands are some of the ways)
```
var myCursor = db.movies.find({
    cast: ["Jeff Bridges", "Tim Robbins"]
})

myCursor.forEach(printjson)
```

```
var myCursor = db.movies.find({
    cast: ["Jeff Bridges", "Tim Robbins"]
})

while (myCursor.hasNext()) {
    printjson(myCursor.next())
}
```

### Projections

- By default _id will be returned in all the projections unless you explicitly specify it.

- will print only the title of the documents matching the specified condition (includes title)
```
db.movies.find({
    genre: "Action, Adventure"
}, {
    title: 1
})
```

- will print all the fields except the title field of the documents matching the specified condition (excludes title)
```
db.movies.find({
    genre: "Action, Adventure"
}, {
    title: 0
})
```

- the below projection will just print titles excluding the _id field as it is specified explicitly.
```
db.movies.find({
    genre: "Action, Adventure"
}, {
    title: 1,
    _id: 0
})
```

### Updating documents

- Update One

```
db.movieDetails.updateOne({
    title: "The Martian"
}, {
    $set: {
        poster: "An image link here"
    }
})
```
```
db.movieDetails.updateOne({
    title: "The Martian"
}, {
    $set: {
        awards: {
            "wins": 8,
            "nominations": 14,
            "text": "Nominated for 3 Golden globes"
        }
    }
})
```

- $inc operator

```
db.movieDetails.updateOne({
    title: "The Martian"
}, {
    $inc: {
        "tomato.reviews": 3,
        "tomato.userReviews": 25
    }
})
```

- $push operator (will push in the array if it exists else it will create an array then push)

```
db.movieDetails.updateOne({
    title: "The Martian"
}, {
    $push: {
        reviews: {
            rating: 4.5,
            date: ISODate("2016-01-12T09:00:00Z"),
            reviewer: "Spencer.H",
            text: "This is sample review"
        }
    }
})
```

will add each object specified in the array as a new array field.
```
db.movieDetails.updateOne({
    title: "The Martian"
}, {
    $push: {
        reviews: {
            $each: [{
                rating: 4.5,
                date: ISODate("2016-01-12T09:00:00Z"),
                reviewer: "Spencer.H",
                text: "This is sample review"
            }, {
                rating: 4.5,
                date: ISODate("2016-01-12T09:00:00Z"),
                reviewer: "Spencer.H",
                text: "This is sample review"
            }, {
                rating: 4.5,
                date: ISODate("2016-01-12T09:00:00Z"),
                reviewer: "Spencer.H",
                text: "This is sample review"
            }]
        }
    }
})
```

- Update many (will modify all the documents matching the filter)

```
db.movieDetails.updateMany({
    rated: null
}, {
    $unset: {
        rated: ""
    }
})
```

- Replace one

The replacement document cannot contain update operators.
replaceOne will apply changes to only one document, the first found in the server that matches the filter expression, using the $natural order of documents in the collection.

```
let filter = {
    title: "A Million Ways to Die in the West"
}

let doc = db.movieDetails.findOne(filter);

doc.poster;

doc.poster = "abc"

doc.genres;

doc.genres.push("TV Series");

db.movieDetails.replaceOne(filter, doc);
```

- Deleting documents

```
db.reviews.deleteOne({
    _id: ObjectId("5d24c0a09789793092daad5c")
})
```

```
db.reviews.deleteOne({
    reviewer_id: 669916896
})
```
