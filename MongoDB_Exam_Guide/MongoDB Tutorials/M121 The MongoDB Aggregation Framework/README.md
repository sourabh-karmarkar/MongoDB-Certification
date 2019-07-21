### Connecting to M121 course atlas cluster using the mongo shell

```
mongo "mongodb://cluster0-shard-00-00-jxeqq.mongodb.net:27017,cluster0-shard-00-01-jxeqq.mongodb.net:27017,cluster0-shard-00-02-jxeqq.mongodb.net:27017/aggregations?replicaSet=Cluster0-shard-0" --authenticationDatabase admin --ssl -u m121 -p aggregations --norc
```



### $match

```
db.solarSystem.aggregate([{
    $match: {
        type: {
            $ne: "Star"
        }
    }
}, {
    $count: "planets"
}])
```
```
// simple first example
db.solarSystem.aggregate([{
    "$match": {
        "atmosphericComposition": {
            "$in": [/O2/]
        },
        "meanTemperature": {
            $gte: -40,
            "$lte": 40
        }
    }
}, {
    "$project": {
        "_id": 0,
        "name": 1,
        "hasMoons": {
            "$gt": ["$numberOfMoons", 0]
        }
    }
}], {
    "allowDiskUse": true
});
```

- Things to remember:
    1) A $match stage may contain a $text query operator, but it must be the first stage in the pipeline.
    2) $match should come early in an aggregation pipeline.
    3) You cannot use $where with match.
    4) $match uses the same query syntax as find.

- Lab - $match

```
var pipeline = {
    $match: {
        "imdb.rating": {
            $gte: 7
        },
        "genres": {
            $nin: ["Crime", "Horror"]
        },
        rated: {
            $in: ["PG", "G"]
        },
        languages: {
            $all: ["English", "Japanese"]
        }
    }
}

db.movies.aggregate(pipeline).itcount()

load('validateLab1.js')

validateLab1(pipeline)
```
Answer is : 15

### $project

```
// project ``name`` and remove ``_id``
db.solarSystem.aggregate([{
    "$project": {
        "_id": 0,
        "name": 1
    }
}]);

// project ``name`` and ``gravity`` fields, including default ``_id``
db.solarSystem.aggregate([{
    "$project": {
        "name": 1,
        "gravity": 1
    }
}]);

// using dot-notation to express the projection fields
db.solarSystem.aggregate([{
    "$project": {
        "_id": 0,
        "name": 1,
        "gravity.value": 1
    }
}]);

// reassing ``gravity`` field with value from ``gravity.value`` embeded field
db.solarSystem.aggregate([{
    "$project": {
        "_id": 0,
        "name": 1,
        "gravity": "$gravity.value"
    }
}]);

// creating a document new field ``surfaceGravity``
db.solarSystem.aggregate([{
    "$project": {
        "_id": 0,
        "name": 1,
        "surfaceGravity": "$gravity.value"
    }
}]);

// creating a new field ``myWeight`` using expressions
db.solarSystem.aggregate([{
    "$project": {
        "_id": 0,
        "name": 1,
        "myWeight": {
            "$multiply": [{
                "$divide": ["$gravity.value", 9.8]
            }, 86]
        }
    }
}]);
```

- Things to remember:
    1) Once we specify one field to retain, we must specify all fields we want to retain. The _id field is the only exception to this.
    2) Beyond simply removing and retaining the fields, $project lets us add new fields.
    3) $project can be used as many times as required within an Aggregation pipeline.
    4) $project can be used to reassign values to existing field names and to derive entirely new fields.

- Lab - Changing document shape with $project

```
var pipeline = [{
    $match: {
        "imdb.rating": {
            $gte: 7
        },
        "genres": {
            $nin: ["Crime", "Horror"]
        },
        rated: {
            $in: ["PG", "G"]
        },
        languages: {
            $all: ["English", "Japanese"]
        }
    }
}, {
    $project: {
        title: 1,
        rated: 1,
        _id: 0
    }
}]

db.movies.aggregate(pipeline);

load('./validateLab2.js')

validateLab2(pipeline)
```
Answer is : 15

- Lab - Changing document shape with $project

```
db.movies.aggregate([{
    $project: {
        _id: 0,
        titleArraySize: {
            $cond: {
                if: {
                    $isArray: {
                        $split: ["$title", " "]
                    }
                },
                then: {
                    $size: {
                        $split: ["$title", " "]
                    }
                },
                else: "NA"
            }
        }
    }
}, {
    $match: {
        titleArraySize: 1
    }
}]).itcount();
```
Answer is : 8068

- Optional Lab - Expressions with $project

    - Let's find how many movies in our movies collection are a "labor of love", where the same person appears in cast, directors, and writers
    - Note that you may have a dataset that has duplicate entries for some films. Don't worry if you count them few times, meaning you should not try to find those duplicates.

```
db.movies.aggregate([{
    $match: {
        cast: {
            $elemMatch: {
                $exists: true
            }
        },
        directors: {
            $elemMatch: {
                $exists: true
            }
        },
        writers: {
            $elemMatch: {
                $exists: true
            }
        }
    }
}, {
    $project: {
        _id: 0,
        cast: {
            $map: {
                input: "$cast",
                as: "castName",
                in: {
                    $arrayElemAt: [{
                        $split: ["$$castName", " ("]
                    }, 0]
                }
            }
        },
        directors: {
            $map: {
                input: "$directors",
                as: "director",
                in: {
                    $arrayElemAt: [{
                        $split: ["$$director", " ("]
                    }, 0]
                }
            }
        },
        writers: {
            $map: {
                input: "$writers",
                as: "writer",
                in: {
                    $arrayElemAt: [{
                        $split: ["$$writer", " ("]
                    }, 0]
                }
            }
        }
    }
}, {
    $project: {
        "labors of love": {
            $gt: [{
                $size: {
                    $setIntersection: ["$cast", "$directors", "$writers"]
                }
            }, 0]
        }
    }
}, {
    $match: {
        "labors of love": true
    }
}, {
    $count: "labors of love"
}])
```
Answer is : 1597

### $addFields stage

- Does not remove fields from the original documents. Instead new transformation fields are appended to the document.
```
db.solarSystem.aggregate([{
    $addFields: {
        _id: 0,
        name: 1,
        gravity: "$gravity.value",
        meanTemperature: 1,
        density: 1,
        mass: "$mass.value",
        radius: "$radius.value",
        sma: "$sma.value"
    }
}])
```

- Using $addFields with project

```
db.solarSystem.aggregate([{
    $project: {
        _id: 0,
        name: 1,
        gravity: 1,
        mass: "$mass.value",
        radius: "$radius.value",
        sma: "$sma.value"
    }
}, {
    $addFields: {
        gravity: "$gravity.value",
        mass: "$mass.value",
        radius: "$radius.value",
        sma: "$sma.value"
    }
}])
```

### $geoNear

- Structure

```
$geoNear:{
    near: <required, the locations to search near>
    distanceField: <required, field to insert in returned documents>
    minDistance: <optional, in meters>
    maxDistance: <optional, in meters>
    query: <optional, allows querying source documents>
    includeLocs: <optional, used to identify which location was used>
    limit: <optional, the maximum number of documents to return>
    num: <optional, same as limit>
    spherical: <optional, required to signal whether using a 2dsphere index>
    distanceMultiplier: <optional, the factor to multiply all distances>
}
```

- query to search for locations near the mongoDB headquarters in New York City

```
db.nycFacilities.aggregate([{
    $geoNear: {
        near: {
            type: "Point",
            coordinates: [-73.98769766092299, 40.757345233626594]
        },
        distanceField: "distanceFromMongoDB",
        spherical: true
    }
}])
```

- Things to remember:
    1) The collection can have one and only one 2dsphere index.
    2) If using 2dsphere the distance is returned in meters. If using legacy coordinates, the distance is returned in radians.
    3) Must be the first stage in the pipeline.
    4) We cannot use the $near predicate in the query field.
    5) $geoNear be used on sharded collections while $near cannot be used.
    6) $geoNear requires that the collections we're performing aggregations on, to have one and only one geo index.

### Cursor-like stages

- Cursor methods
    - Skip: Skip the first N documents. (skipped documents are calculated based on the insertion order.)

    ```
    db.solarSystem.find({}, {
        _id: 0,
        name: 1,
        numberOfMoons: 1
    }).skip(5)
    ```

    - Limit: Display the first N documents. (calculated based on the insertion order.)

    ```
    db.solarSystem.find({}, {
        _id: 0,
        name: 1,
        numberOfMoons: 1
    }).limit(5)
    ```

    - Sort: Sort documents (-1 ---> descending, 1 ---> ascending)

    ```
    db.solarSystem.find({}, {
        _id: 0,
        name: 1,
        numberOfMoons: 1
    }).sort({
        numberOfMoons: -1
    })
    ```

- Cursor stages
    - $skip: Skip the first N documents. (skipped documents are calculated based on the insertion order.)

    ```
    db.solarSystem.aggregate([{
        $project: {
            _id: 0,
            name: 1,
            numberOfMoons: 1
        }
    }, {
        "$skip": 1
    }])
    ```

    - $limit: Display the first N documents. (calculated based on the insertion order.)

    ```
    db.solarSystem.aggregate([{
        $project: {
            _id: 0,
            name: 1,
            numberOfMoons: 1
        }
    }, {
        "$limit": 5
    }])
    ```

    - $count: Counts the number of returned documents

    ```
    db.solarSystem.aggregate([{
        $match: {
            type: "Terrestrial planet"
        }
    }, {
        $project: {
            _id: 0,
            name: 1,
            numberOfMoons: 1
        }
    }, {
        $count: "terrestrial planets"
    }])
    ```

    - $sort: Sort documents (-1 ---> descending, 1 ---> ascending)
        1) can also be used with multiple fields
        2) If sort is near the beginning of our pipeline, in place before a project, and unwinds in the group stage, it can take advantage of indexes.
        3) Otherwise, this sort stage will perform an in-memory sort, , which will greatly increase the memory consumption of the server.
        4) Sort operations within that vision pipeline are limited to 100 megabytes of RAM by default.
        5) To allow handling larger data sets, we need to allow DiskUse, which is an aggregation pipeline option that we can provide to the aggregate function.
        6) By doing so, we will be performing the excess of 100 megabytes of memory required to do a sort using disk to help us sort out the results.

    ```
    db.solarSystem.aggregate([{
        $project: {
            _id: 0,
            name: 1,
            numberOfMoons: 1
        }
    }, {
        "$sort": {
            hasMagneticField: -1,
            numberOfMoons: -1
        }
    }])
    ```

    ```
    db.solarSystem.aggregate([{
        $project: {
            _id: 0,
            name: 1,
            numberOfMoons: -1,
            hasMagneticField: -1
        }
    }, {
        "$sort": {
            hasMagneticField: -1,
            numberOfMoons: -1
        }
    }], {
        allowDiskUse: true
    })
    ```

### $sample

- Randomly selects the specified number of documents from its input
- There are 2 methods for $sample
    - $sample is the first stage of the pipeline
    - N is less than 5% of the total documents in the collection
    - The collection contains more than 100 documents
- If all the following conditions are met, $sample uses a pseudo-random cursor to select documents
- If any of the above conditions are NOT met, $sample performs a collection scan followed by a random sort to select N documents. In this case, the $sample stage is subject to the sort memory restrictions.

#### ---> Lab: Using Cursor-like stages

MongoDB has another movie night scheduled. This time, we polled employees for their favorite actress or actor, and got these results

favorites = [
    "Sandra Bullock",
    "Tom Hanks",
    "Julia Roberts",
    "Kevin Spacey",
    "George Clooney"
]

For movies released in the USA with a tomatoes.viewer.rating greater than or equal to 3, calculate a new field called num_favs that represets how many favorites appear in the cast field of the movie.
Sort your results by num_favs, tomatoes.viewer.rating, and title, all in descending order.
What is the title of the 25th film in the aggregation result?

```
var favorites = ["Sandra Bullock", "Tom Hanks", "Julia Roberts", "Kevin Spacey", "George Clooney"];
db.movies.aggregate([{
    $match: {
        "tomatoes.viewer.rating": {
            $gte: 3
        },
        countries: "USA",
        cast: {
            $elemMatch: {
                $exists: true
            }
        }
    }
}, {
    $project: {
        _id: 0,
        title: 1,
        "tomatoes.viewer.rating": 1,
        num_favs: {
            $size: {
                $setIntersection: [
                    "$cast",
                    favorites
                ]
            }
        }
    }
}, {
    $match: {
        num_favs: {
            $gte: 1
        }
    }
}, {
    $sort: {
        num_favs: -1,
        "tomatoes.viewer.rating": -1,
        title: -1
    }
}, {
    $skip: 24
}, {
    $limit: 1
}])
```
Answer is : "The Heat"

#### ---> Lab: Bringing it all together

Calculate an average rating for each movie in our collection where English is an available language, the minimum imdb.rating is at least 1, the minimum imdb.votes is at least 1, and it was released in 1990 or after. You'll be required to rescale (or normalize) imdb.votes. The formula to rescale imdb.votes and calculate normalized_rating is included as a handout. (scaling.js)

What film has the lowest normalized_rating?

```
var x_max = 1521105
var x_min = 5
var min = 1
var max = 10

db.movies.aggregate([{
    $match: {
        languages: {
            $in: ["English"]
        },
        "imdb.rating": {
            $gte: 1
        },
        "imdb.votes": {
            $gte: 1
        },
        year: {
            $gte: 1990
        }
    }
}, {
    $project: {
        _id: 0,
        title: 1,
        "imdb.rating": 1,
        "imdb.votes": 1,
        normalized_rating: {
            $avg: ["$imdb.rating", {
                $add: [min, {
                    $multiply: [max - min, {
                        $divide: [{
                            $subtract: ["$imdb.votes", x_min]
                        }, {
                            $subtract: [x_max, x_min]
                        }]
                    }]
                }]
            }]
        },
    }
}, {
    $sort: {
        normalized_rating: 1
    }
}, {
    $limit: 1
}])
```
Answer is : The Christmas Tree
