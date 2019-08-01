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

### The $group stage

```
db.movies.aggregate([{
    $group: {
        _id: "$year"
    }
}])
```

```
db.movies.aggregate([{
    $group: {
        _id: "$year",
        num_of_films: {
            $sum: 1
        }
    }
}])
```

```
db.movies.aggregate([{
    $group: {
        _id: "$year",
        num_of_films: {
            $sum: 1
        }
    }
}, {
    $sort: {
        num_of_films: -1
    }
}])
```

```
db.movies.aggregate([{
        $group: {
            _id: {
                numDirectors: {
                    $cond: [{
                            $isArray: "$directors"
                        }, {
                            $size: "$directors"
                        },
                        0
                    ]
                }
            },
            numFilms: {
                $sum: 1
            },
            averageMetacritic: {
                $avg: "$metacritic"
            },
        }
    },
    {
        $sort: {
            "_id.numDirectors": -1
        }
    }
])
```

```
db.movies.aggregate([{
    $group: {
        _id: null,
        "count": {
            $sum: 1
        }
    }
}])
```

```
db.movies.aggregate([{
    $match: {
        metacritic: {
            $gte: 0
        }
    }
}, {
    $group: {
        _id: null,
        averageMetaCritic: {
            $avg: "$metacritic"
        }
    }
}])
```

- Things to remember:
    1) _id is where to specify what incoming documents should be grouped on
    2) Can use all accumulator expressions within $group
    3) $group can be used multiple times within a pipeline
    4) It may be necessary to sanitize incoming data.

### Accumulator Expressions with $project

```
db.icecream_data.aggregate([{
    $project: {
        _id: 0,
        max_high: {
            $reduce: {
                input: "$trends",
                initialValue: -Infinity,
                in: {
                    $cond: [{
                        $gt: ["$$this.avg_high_tmp", "$$value"]
                    }, "$$this.avg_high_tmp", "$$value"]
                }
            }
        }
    }
}])
```

```
db.icecream_data.aggregate([{
    $project: {
        _id: 0,
        max_high: {
            $max: "$trends.avg_high_tmp"
        }
    }
}])
```

```
db.icecream_data.aggregate([{
    $project: {
        _id: 0,
        max_low: {
            $min: "$trends.avg_low_tmp"
        }
    }
}])
```

```
db.icecream_data.aggregate([{
    $project: {
        _id: 0,
        average_cpi: {
            $avg: "$trends.icecream_cpi"
        },
        cpi_deviation: {
            $stdDevPop: "$trends.icecream_cpi"
        }
    }
}])
```

```
db.icecream_data.aggregate([{
    $project: {
        _id: 0,
        "yearly_sales (millions)": {
            $sum: "$trends.icecream_sales_in_millions"
        }
    }
}])
```

- Things to remember:
    1) Available Accumulator Expressions : $sum, $avg, $max, $min, $stdDevPop, $stdDevSam
    2) Expressions have no memory between documents
    3) May still have to use $reduce or $map for more complex calculations

- Lab - $group and Accumulators

```
db.movies.aggregate([{
    $match: {
        awards: {
            $regex: /Won \d{1,2} Oscars?/
        }
    }
}, {
    $group: {
        _id: null,
        highest_rating: {
            $max: "$imdb.rating"
        },
        lowest_rating: {
            $min: "$imdb.rating"
        },
        average_rating: {
            $avg: "$imdb.rating"
        },
        deviation: {
            $stdDevSamp: "$imdb.rating"
        }
    }
}])
```

### The $unwind stage

```
db.movies.aggregate([{
        $match: {
            "imdb.rating": {
                $gt: 0
            },
            year: {
                $gte: 2010,
                $lte: 2015
            },
            runtime: {
                $gte: 90
            }
        }
    }, {
        $unwind: "$genres"
    },
    {
        $group: {
            _id: {
                year: "$year",
                genre: "$genres"
            },
            average_rating: {
                $avg: "$imdb.rating"
            }
        }
    },
    {
        $sort: {
            "_id.year": -1,
            average_rating: -1
        }
    },
    {
        $group: {
            _id: "$_id.year",
            genre: {
                $first: "$_id.genre"
            },
            average_rating: {
                $first: "$average_rating"
            }
        }
    },
    {
        $sort: {
            _id: -1
        }
    }
])
```

- Things to remember:
    1) $unwind only works on array values
    2) There are 2 forms for unwind, short form and long form
    3) Using unwind on large collections with big documents may lead to performance issues

- Lab - $unwind

```
db.movies.aggregate([{
        $match: {
            languages: "English"
        }
    },
    {
        $project: {
            _id: 0,
            cast: 1,
            "imdb.rating": 1
        }
    },
    {
        $unwind: "$cast"
    }, {
        $group: {
            _id: "$cast",
            numFilms: {
                $sum: 1
            },
            average: {
                $avg: "$imdb.rating"
            }
        }
    },
    {
        $project: {
            numFilms: 1,
            average: {
                $divide: [{
                    $trunc: {
                        $multiply: ["$average", 10]
                    }
                }, 10]
            }
        }
    }, {
        $sort: {
            numFilms: -1
        }
    },
    {
        $limit: 1
    }
])
```

### The $lookup stage

```
db.air_alliances.aggregate([{
    $lookup: {
        from: "air_airlines",
        localField: "airlines",
        foreignField: "name",
        as: "airlines"
    }
}])
```

- Things to remember:
    1) The from collection cannot be sharded
    2) The from collection must be in the same database
    3) The values in the localField and foreignField are matched on equality
    4) as can be any name, but if it exists in the working document that field will be overwritten

- Lab - Using $lookup

Which alliance from air_alliances flies the most routes with either a Boeing 747 or an Airbus A380 (abbreviated 747 and 380 in air_routes)?

```
db.air_routes.aggregate([{
    $match: {
        airplane: /747|380/
    }
}, {
    $lookup: {
        from: "air_alliances",
        foreignField: "airlines",
        localField: "airline.name",
        as: "alliance"
    }
}, {
    $unwind: "$alliance"
}, {
    $group: {
        _id: "$alliance.name",
        count: {
            $sum: 1
        }
    }
}, {
    $sort: {
        count: -1
    }
}, {
    $limit: 1
}])
```

Answer is : { "_id" : "SkyTeam", "count" : 16 }

### $graphLookup : Introduction

- Finding all the persons that report to the CTO directly or indirectly

```
db.parent_reference.aggregate([{
    $match: {
        name: "Eliot"
    }
}, {
    $graphLookup: {
        from: "parent_reference",
        startWith: "$_id",
        connectFromField: "_id",
        connectToField: "reports_to",
        as: "all_reports"
    }
}])
```

- Finding the chain uptill the root element from a specified element.

```
db.parent_reference.aggregate([{
    $match: {
        name: "Shannon"
    }
}, {
    $graphLookup: {
        from: "parent_reference",
        startWith: "$reports_to",
        connectFromField: "reports_to",
        connectToField: "_id",
        as: "bosses"
    }
}])
```

### $graphLookup : Simple Lookup Reverse Schema

```
db.child_reference.aggregate([{
    $match: {
        name: "Dev"
    }
}, {
    $graphLookup: {
        from: "child_reference",
        startWith: "$direct_reports",
        connectFromField: "direct_reports",
        connectToField: "name",
        as: "all_reports"
    }
}])
```

### $graphLookup : maxDepth and depthField

```
db.child_reference.aggregate([{
    $match: {
        name: "Dev"
    }
}, {
    $graphLookup: {
        from: "child_reference",
        startWith: "$direct_reports",
        connectFromField: "direct_reports",
        connectToField: "name",
        as: "till_2_level_reports",
        maxDepth: 1,
        depthField: "level"
    }
}])
```

### $graphLookup : Cross Collection Lookup

```
db.air_airlines.aggregate([{
    $match: {
        name: "TAP Portugal"
    }
}, {
    $graphLookup: {
        from: "air_routes",
        as: "chain",
        startWith: "$base",
        connectFromField: "dst_airport",
        connectToField: "src_airport",
        maxDepth: 1,
        restrictSearchWithMatch: {
            "airline.name": "TAP Portugal"
        }
    }
}])
```

- Lab: $graphLookup

```
db.air_alliances.aggregate([{
        $match: {
            name: "OneWorld"
        }
    },
    {
        $graphLookup: {
            startWith: "$airlines",
            from: "air_airlines",
            connectFromField: "name",
            connectToField: "name",
            as: "airlines",
            maxDepth: 0,
            restrictSearchWithMatch: {
                country: {
                    $in: ["Germany", "Spain", "Canada"]
                }
            }
        }
    },
    {
        $graphLookup: {
            startWith: "$airlines.base",
            from: "air_routes",
            connectFromField: "dst_airport",
            connectToField: "src_airport",
            as: "connections",
            maxDepth: 1
        }
    },
    {
        $project: {
            validAirlines: "$airlines.name",
            "connections.dst_airport": 1,
            "connections.airline.name": 1
        }
    },
    {
        $unwind: "$connections"
    },
    {
        $project: {
            isValid: {
                $in: ["$connections.airline.name", "$validAirlines"]
            },
            "connections.dst_airport": 1
        }
    },
    {
        $match: {
            isValid: true
        }
    },
    {
        $group: {
            _id: "$connections.dst_airport"
        }
    }
])
```

### Facets : Single Facet Query

- Load the companies.json in handouts to the your mongodb atlas cluster

```
db.companies.createIndex({
    description: "text",
    overview: "text"
})
```
```
db.companies.aggregate([{
    $match: {
        $text: {
            $search: "network"
        }
    }
}])
```
```
db.companies.aggregate([{
    $match: {
        $text: {
            $search: "network"
        }
    }
}, {
    $sortByCount: "$category_code"
}])
```
```
db.companies.aggregate([{
    $match: {
        $text: {
            $search: "network"
        }
    }
}, {
    $unwind: "$offices"
}, {
    $match: {
        "offices.city": {
            $ne: ""
        }
    }
}, {
    $sortByCount: "$offices.city"
}])
```

### The $bucket Stage

```
db.movies.aggregate([{
    $bucket: {
        groupBy: "$imdb.rating",
        boundaries: [0, 5, 8, Infinity],
        default: "not rated"
    }
}])
```

```
db.movies.aggregate([{
    $bucket: {
        groupBy: "$imdb.rating",
        boundaries: [0, 5, 8, Infinity],
        default: "not rated",
        output: {
            average_per_bucket: {
                $avg: "$imdb.rating"
            },
            count: {
                $sum: 1
            }
        }
    }
}])
```

### Facets: Manual Buckets

```
db.companies.aggregate([{
    $match: {
        founded_year: {
            $gt: 1980
        },
        number_of_employees: {
            $ne: null
        }
    }
}, {
    $bucket: {
        groupBy: "$number_of_employees",
        boundaries: [0, 20, 50, 100, 500, 1000, Infinity]
    }
}])
```

```
db.col.insert({x : 'a'})

db.col.aggregate([{
    $bucket: {
        groupBy: "$x",
        boundaries: [0, 50, 100]
    }
}])

db.col.aggregate([{
    $bucket: {
        groupBy: "$x",
        boundaries: [0, 50, 100],
        default: "Other"
    }
}])
```

```
db.companies.aggregate([{
    $match: {
        founded_year: {
            $gt: 1980
        }
    }
}, {
    $bucket: {
        groupBy: "$number_of_employees",
        boundaries: [0, 20, 50, 100, 500, 1000, Infinity],
        default: "Other"
    }
}])
```

```
db.companies.aggregate([{
    $match: {
        founded_year: {
            $gt: 1980
        }
    }
}, {
    $bucket: {
        groupBy: "$number_of_employees",
        boundaries: [0, 20, 50, 100, 500, 1000, Infinity],
        default: "Other",
        output: {
            total: {
                $sum: 1
            },
            average: {
                $avg: "$number_of_employees"
            },
            categories: {
                $addToSet: "$category_code"
            }
        }
    }
}])
```

- Things to remember
    1) Must always specify atleast 2 values to boundaries
    2) boundaries must all be of the same general type (Numeric, String)
    3) count is inserted by default with no output, but removed when output is specified

# The $bucketAuto Stage

```
db.movieDetails.aggregate([{
    $match: {
        "imdb.rating": {
            $gte: 0
        }
    }
}, {
    $bucketAuto: {
        groupBy: "$imdb.rating",
        buckets: 4,
        output: {
            average_per_bucket: {
                $avg: "$imdb.rating"
            },
            count: {
                $sum: 1
            }
        }
    }
}])
```

```
use agg

function make_granularity_values() {
    for (let i = 0; i < 100; i++) {
        db.granularity_test.insertOne({
            powers_of_2: Math.pow(2, Math.floor(Math.random() * 10)),
            renard_and_e: Math.random() * 10
        })
    }
}

make_granularity_values()

db.granularity_test.count()

db.granularity_test.aggregate([{
    $bucketAuto: {
        groupBy: "$powers_of_2",
        buckets: 10,
        granularity: "POWERSOF2"
    }
}])
```

### Facets: Auto Buckets

```
db.companies.aggregate([{
    $match: {
        "offices.city": "New York"
    }
}, {
    $bucketAuto: {
        groupBy: "$founded_year",
        buckets: 5,
        output: {
            total: {
                $sum: 1
            },
            average: {
                $avg: "$number_of_employees"
            }
        }
    }
}])

for (i = 1; i <= 1000; i++) {
    db.series.insert({
        _id: i
    })
}

db.series.aggregate([{
    $bucketAuto: {
        groupBy: "$_id",
        buckets: 5
    }
}])

db.series.aggregate([{
    $bucketAuto: {
        groupBy: "$_id",
        buckets: 5,
        granularity: "R20"
    }
}])
```

- Things to remember
    1) Cardinality of groupBy expression may impact even distribution and number of buckets
    2) Specifying a granularity requires the expression to groupBy to resolve to a numeric value

### Facets: Multiple Facets

```
db.companies.aggregate([{
    $match: {
        $text: {
            $search: "Databases"
        }
    }
}, {
    $facet: {
        Categories: [{
            "$sortByCount": "$category_code"
        }],
        Employees: [{
                $match: {
                    founded_year: {
                        $gt: 1980
                    }
                }
            },
            {
                $bucket: {
                    groupBy: "$number_of_employees",
                    boundaries: [0, 20, 50, 100, 500, 1000, Infinity],
                    default: "Other"
                }
            }
        ],
        Founded: [{
            $match: {
                "offices.city": "New York"
            }
        }, {
            $bucketAuto: {
                groupBy: "$founded_year",
                buckets: 5
            }
        }]
    }
}])
```

- Lab - $facets

How many movies are in both the top ten highest rated movies according to the imdb.rating and the metacritic fields? We should get these results with exactly one access to the database.

```
db.movies.aggregate([{
        $match: {
            metacritic: {
                $gte: 0
            },
            "imdb.rating": {
                $gte: 0
            }
        }
    },
    {
        $project: {
            _id: 0,
            metacritic: 1,
            imdb: 1,
            title: 1
        }
    },
    {
        $facet: {
            top_metacritic: [{
                    $sort: {
                        metacritic: -1,
                        title: 1
                    }
                },
                {
                    $limit: 10
                },
                {
                    $project: {
                        title: 1
                    }
                }
            ],
            top_imdb: [{
                    $sort: {
                        "imdb.rating": -1,
                        title: 1
                    }
                },
                {
                    $limit: 10
                },
                {
                    $project: {
                        title: 1
                    }
                }
            ]
        }
    },
    {
        $project: {
            movies_in_both: {
                $setIntersection: ["$top_metacritic", "$top_imdb"]
            }
        }
    }
])
```

### The $sortByCount Stage

```
// We use the below query to sort and count the number of documents without $sortByCount
db.movies.aggregate([{
    $group: {
        _id: "$imdb.rating",
        count: {
            $sum: 1
        }
    }
}, {
    $sort: {
        count: -1
    }
}])

// We use the below query to sort and count the number of documents using $sortByCount
db.movies.aggregate([{
    $sortByCount: "$imdb.rating"
}])
```

- Things to remember
    1) is equivalent to a group stage to count occurence, and then sorting in the descending order

### The $redact Stage

Helps protect information from unauthorized access
- $$DESCEND - Retain this level
- $$PRUNE - Remove
- $$KEEP - Retain

```
db.employees.aggregate([{
    $redact: {
        $cond: [{
            $in: ["Management", "$acl"]
        }, "$$DESCEND", "$$PRUNE"]
    }
}])
```

- Things to remember
    1) $$KEEP and $$PRUNE automatically apply to all levels below the evaluated level
    2) $$DESCEND retains the current level and evaluates the next level down
    3) $redact is not for restricting access to a collection

### The $out Stage

- Takes the documents returned by the aggregation pipeline and writes them to a specified collection. The $out operator must be the last stage in the pipeline and cannot be used within a facet

```
// List the company having maximum number of offices and write it as a collection

db.companies.aggregate([{
    $project: {
        _id: 0,
        name: 1,
        office_size: {
            $size: "$offices"
        }
    }
}, {
    $sort: {
        office_size: -1
    }
}, {
    $limit: 1
}, {
    $out: "maximumOfficesCompany"
}])
```

- Things to remember
    1) Will create a new collection or overwrite an existing collection if specified
    2) Honors indexes on existing collections
    3) Will not create or overwrite data if pipeline errors
    4) Created collection in the same database as the source collection

### Views

```
db.createView("bronze_banking", "customers", [{
    $match: {
        accountType: "bronze"
    },
    $project: {
        _id: 0,
        name: {
            $concat: [{
                $cond: [{
                    $eq: ["$gender", "female"]
                }, "Miss", "Mr."]
            }, " ", "$name.first", " ", "$name.last"]
        },
        phone: 1,
        email: 1,
        address: 1,
        account_editing: {
            $substr: ["$accountNumber", 7, -1]
        }
    }
}])

// getting all collections in a database and seeing their information
db.getCollectionInfos()

// getting information on views only
db.system.views.find()
```

- Some restrictions on views
    1) No write operations
    2) No index operations (create, update)
    3) No renaming
    4) No mapReduce
    5) No $text
    6) No geoNear or $geoNear
    7) Collation restrictions
    8) find() operation with projection operators are not permitted.
        - $
        - $elemMatch
        - $slice
        - $meta

- Things to remember
    1) Views contain no data themselves. They are created on demand and reflect the data in the source collection.
    2) Views are read only. Write operations to views will error.
    3) Views have some restrictions. They must abide by the rules of the Aggregation Framework, and cannot contain the find() projection operators.
    4) Horizontal slicing is performed with the $match stage, reducing the number of documents that are returned.
    5) Vertical slicing is performed with a $project or other shaping stage, modifying individual documents.