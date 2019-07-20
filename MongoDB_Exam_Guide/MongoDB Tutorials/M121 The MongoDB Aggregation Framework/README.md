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
