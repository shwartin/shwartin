# Mongodb quick start
in macOS


## install and run

start mongodb
```
brew services start mongodb-community@5.0
```

stop
```
brew services stop mongodb-community@5.0
```


## shells
- Mongodb plugin for vscode. Create new playground in vscode to run.
- Robo 3t

**howto increase font size in Robo 3t?**

In file `/Users/<user>/.3T/robo-3t/1.4.4/robo3t.json` change `"textFontPointSize": -1` -> `"textFontPointSize": 15`


## aggregation operators
- $match — filter queries
- $group
- $count
- $sort
- $project — filter fields
- $limit
- $unwind — split each document with array to several documents
- $skip
- $out — result of the aggregation to another collection


## accumulator operators 
Also known as accumulators. Usually used in the `$group` stage:

- $sum
- $avg
- $max
- $min


## unary operators
Usually used in the `$project` stage.  In the `$group` stage used only in conjunction with accumulators.

- $and
- $type — BSON type of the filed's value
- $or
- $multiply
- $gt — greater then
- $lt — leater then
- $ne — not equal

Sample documents here: https://github.com/bstashchuk/MongoDB-Aggregation-Tutorial


## $match
```
db.persons
    .aggregate([
        {$match: {tags: {$size: 3}}}
    ])
```
is equal
```
db.persons.find({tags: {$size: 3}})

```

result:
```
/* 1 */
{
    "_id" : ObjectId("62416ea2c2705ea08f62438c"),
    "index" : 3,
    "name" : "Karyn Rhodes",
    "isActive" : true,
    "registered" : ISODate("2014-03-11T03:02:33.000Z"),
    "age" : 39,
    "gender" : "female",
    "eyeColor" : "green",
    "favoriteFruit" : "strawberry",
    "company" : {
        "title" : "RODEMCO",
        "email" : "karynrhodes@rodemco.com",
        "phone" : "+1 (801) 505-3760",
        "location" : {
            "country" : "USA",
            "address" : "521 Seigel Street"
        }
    },
    "tags" : [ 
        "cillum", 
        "exercitation", 
        "excepteur"
    ]
}

/* 2 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624391"),
    "index" : 8,
    "name" : "Dale Holman",
    "isActive" : false,
    "registered" : ISODate("2014-07-11T09:08:36.000Z"),
    "age" : 22,
    "gender" : "male",
    "eyeColor" : "green",
    "favoriteFruit" : "strawberry",
    "company" : {
        "title" : "ISONUS",
        "email" : "daleholman@isonus.com",
        "phone" : "+1 (871) 452-3036",
        "location" : {
            "country" : "Italy",
            "address" : "586 Blake Court"
        }
    },
    "tags" : [ 
        "tempor", 
        "aliqua", 
        "duis"
    ]
}

...
```


## $group
`_id` is mandatory filed. 

Get distinct age field and set as `_id`
```
db.persons.aggregate([
    { $group: { _id: "$age" } }
])
```

Result unsorted:
```
/* 1 */
{
    "_id" : 22
}

/* 2 */
{
    "_id" : 32
}

...
```

`_id` field with two documents
```
db.persons.aggregate([
    { $group: { _id: { age: "$age", gender: "$gender" } } }
])
```

Result is all unique combinations:
```
/* 1 */
{
    "_id" : {
        "age" : 40,
        "gender" : "male"
    }
}

/* 2 */
{
    "_id" : {
        "age" : 25,
        "gender" : "male"
    }
}

...
```

Group with nested fields:
```
db.persons.aggregate([
    { $group: { _id: "$company.location.country" } }
])
```

Result:
```
/* 1 */
{
    "_id" : "Italy"
}

/* 2 */
{
    "_id" : "France"
}

/* 3 */
{
    "_id" : "USA"
}

/* 4 */
{
    "_id" : "Germany"
}
```


## $match -> $group
Use `$match` and then `$group` more effective than first `$group` and then `$match`.

```
db.persons.aggregate([
    { $match: {gender: "female"}},
    { $group: {_id: {age: "$age", eyeColor: "$eyeColor" }}}
])
```

Result:
```
/* 1 */
{
    "_id" : {
        "age" : 23,
        "eyeColor" : "brown"
    }
}

/* 2 */
{
    "_id" : {
        "age" : 22,
        "eyeColor" : "blue"
    }
}

...
```

Reverse $group and $match
```
db.persons.aggregate([
    { $group: {_id: {age: "$age", eyeColor: "$eyeColor" }}},
    { $match: {gender: "female"}}
])
```

Result:
```
Fetched 0 record(s) in 0ms
```


## $group -> $mach
```
db.persons.aggregate([
    { $group: {_id: {age: "$age", eyeColor: "$eyeColor" }}},
    { $match: {"_id.eyeColor": "blue"}}
])
```

```
/* 1 */
{
    "_id" : {
        "age" : 38,
        "eyeColor" : "blue"
    }
}

/* 2 */
{
    "_id" : {
        "age" : 33,
        "eyeColor" : "blue"
    }
}

...
```


## $count
`$count` is always the last one on aggregation.

```
db.persons.aggregate([
    {$count: "allDocumentsCount"}
])
```

Result:
```
/* 1 */
{
    "allDocumentsCount" : 1000
}
```


## different count methods with time execution

Client-side

`db.persons.aggregate([]).toArray().lenght` -> `1000` 1.7 sec

`db.persons.aggregate([]).itcount` -> `1000` 1,4 sec


Server-side

`db.persons.aggregate([{$count: "total"}])` -> `{"total": 1000}`  0,21 sec

`db.persons.find({}).count()` -> `1000`  0,21 sec (wrapper of the previous)


## $group -> $count
Example 1
```
db.persons.aggregate([
    {$group: {_id: "$eyeColor"}},
    {$count: "eyeColor"}
])
```

```
/* 1 */
{
    "eyeColor" : 3
}
```

Example 2
```
db.persons.aggregate([
    {$group: {_id: {eyeColor: "$eyeColor", age: "$age"}}},
    {$count: "eyeColorAndAge"}
])
```

```
/* 1 */
{
    "eyeColorAndAge" : 63
}
```


## $sort
-1 ascending
1 descending

```
db.persons.aggregate([
    {$sort: {age: 1}}
])
```

```
/* 1 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
	...
	"age" : 20,
	...
}

/* 2 */
{
    "_id" : ObjectId("62416ea2c2705ea08f62438e"),
	...
	"age" : 20,
	...
}

...

```


```
db.persons.aggregate([
    {$sort: {age: 1, gender: -1, eyeColor: -1}}
])
```

```
try it out
```


## ($match ->) $group -> $sort
One filed
```
db.persons.aggregate([
    {$group: {_id: "$eyeColor"}},
    {$sort: {_id: 1}}
])
```

```
/* 1 */
{
    "_id" : "blue"
}

/* 2 */
{
    "_id" : "brown"
}

/* 3 */
{
    "_id" : "green"
}
```

Two fields
```
db.persons.aggregate([
    {$group: {_id: {eyeColor: "$eyeColor", favoriteFruit: "$favoriteFruit"}}},
    {$sort: {"_id.eyeColor": 1, "_id.favoriteFruit": -1}}
])
```

```
/* 1 */
{
    "_id" : {
        "eyeColor" : "blue",
        "favoriteFruit" : "strawberry"
    }
}

/* 2 */
{
    "_id" : {
        "eyeColor" : "blue",
        "favoriteFruit" : "banana"
    }
}

/* 3 */
{
    "_id" : {
        "eyeColor" : "blue",
        "favoriteFruit" : "apple"
    }
}

...
```

Adding `$match`
```
db.persons.aggregate([
    {$match: {eyeColor: {$ne: "blue"}}},
    {$group: {_id: {eyeColor: "$eyeColor", favoriteFruit: "$favoriteFruit"}}},
    {$sort: {"_id.eyeColor": 1, "_id.favoriteFruit": -1}}
])
```

```
try it out
```


## $project

```
db.persons.aggregate([
    {$project: {name: 1, "company.title": 1}}
])
```

```
/* 1 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
    "name" : "Aurelia Gonzales",
    "company" : {
        "title" : "YURTURE"
    }
}

/* 2 */
{
    "_id" : ObjectId("62416ea2c2705ea08f62438a"),
    "name" : "Kitty Snow",
    "company" : {
        "title" : "DIGITALUS"
    }
}

...
```


```
db.persons.aggregate([
    {$project: {_id: 0, name: 1, "company.title": 1}}
])
```

```
/* 1 */
{
    "name" : "Aurelia Gonzales",
    "company" : {
        "title" : "YURTURE"
    }
}

/* 2 */
{
    "name" : "Kitty Snow",
    "company" : {
        "title" : "DIGITALUS"
    }
}

...
```


```
db.persons.aggregate([
    {$project: {eyeColor: 0, age: 0}}
])
```

```
try it out
```


```
db.persons.aggregate([
    {$project: {name: 1, newAge: "$age"}}
])
```

```
/* 1 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
    "name" : "Aurelia Gonzales",
    "newAge" : 20
}

/* 2 */
{
    "_id" : ObjectId("62416ea2c2705ea08f62438a"),
    "name" : "Kitty Snow",
    "newAge" : 38
}

...
```


## $project with new fields
Usualy used to restructure document.

```
db.persons.aggregate([
    {$project: {
        _id: 0,
        index: 1,
        name: 1,
        info: {
            eyes: "$eyeColor",
            company: "$company.title",
            country: "$company.location.country"
        }
    }}
])
```

```
/* 1 */
{
    "index" : 0,
    "name" : "Aurelia Gonzales",
    "info" : {
        "eyes" : "green",
        "company" : "YURTURE",
        "country" : "USA"
    }
}

/* 2 */
{
    "index" : 1,
    "name" : "Kitty Snow",
    "info" : {
        "eyes" : "blue",
        "company" : "DIGITALUS",
        "country" : "Italy"
    }
}

...
```


## $limit -> $match -> $group

Get first 100 documets in the collection and do some logic:
```
db.persons.aggregate([
    {$limit: 100},
    {$match: {eyeColor: {$ne: "blue"}}},
    {$group: {_id: {eyeColor: "$eyeColor", favoriteFruit: "$favoriteFruit"}}},
    {$sort: {"_id.eyeColor": 1, "_id.favoriteFruit": -1}}
])

```

```
/* 1 */
{
    "_id" : {
        "eyeColor" : "brown",
        "favoriteFruit" : "strawberry"
    }
}

/* 2 */
{
    "_id" : {
        "eyeColor" : "brown",
        "favoriteFruit" : "banana"
    }
}

...
```


## arrays $group 
```
db.persons.aggregate([
    {$group: {_id: "$tags"}},
])
```

```
/* 1 */
{
    "_id" : [ 
        "ad", 
        "ea", 
        "do", 
        "pariatur"
    ]
}

/* 2 */
{
    "_id" : [ 
        "eu", 
        "dolor", 
        "culpa", 
        "exercitation", 
        "aute"
    ]
}

...
```


## $unwind
```
db.persons.aggregate([
    {$unwind: "$tags"}
])
```

```
/* 1 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
	...
    "tags" : "enim"
}

/* 2 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
	...
	"tags" : "id"
}

/* 3 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
	...
    "tags" : "velit"
}

...
```


## $unwind -> $project

```
db.persons.aggregate([
    {$unwind: "$tags"},
    {$project: {name: 1, index: 1, tags: 1}}
])
```

```
/* 1 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
    "index" : 0,
    "name" : "Aurelia Gonzales",
    "tags" : "enim"
}

/* 2 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
    "index" : 0,
    "name" : "Aurelia Gonzales",
    "tags" : "id"
}

/* 3 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
    "index" : 0,
    "name" : "Aurelia Gonzales",
    "tags" : "velit"
}

...
```


## $unwind -> $group
```
db.persons.aggregate([
    {$unwind: "$tags"},
    {$group: {_id: "$tags"}}
])
```

```
/* 1 */
{
    "_id" : "sit"
}

/* 2 */
{
    "_id" : "ut"
}

/* 3 */
{
    "_id" : "est"
}

...
```


## $sum and $group
```
db.persons.aggregate([
    {$group: {
        _id: "$age",
        count: {$sum: 1}
     }}
])
```

```
/* 1 */
{
    "_id" : 40,
    "count" : 38.0
}

/* 2 */
{
    "_id" : 21,
    "count" : 58.0
}

/* 3 */
{
    "_id" : 26,
    "count" : 51.0
}

...
```


## $sum, $unwind and $group

NumberInt to get integers
```
db.persons.aggregate([
    {$unwind: "$tags"},
    {$group: {
        _id: "$tags",
        count: {$sum: NumberInt(1)}
     }}
])
```

```
/* 1 */
{
    "_id" : "Lorem",
    "count" : 48
}

/* 2 */
{
    "_id" : "nulla",
    "count" : 65
}

/* 3 */
{
    "_id" : "eu",
    "count" : 56
}

...
```


## $avg
```
db.persons.aggregate([
    {$group: {
        _id: "$company.location.country",
        avgAge: {$avg: "$age"}
     }}
])
```

```
/* 1 */
{
    "_id" : "Italy",
    "avgAge" : 29.2719665271967
}

/* 2 */
{
    "_id" : "France",
    "avgAge" : 29.9959183673469
}

/* 3 */
{
    "_id" : "USA",
    "avgAge" : 30.3254901960784
}

/* 4 */
{
    "_id" : "Germany",
    "avgAge" : 29.72030651341
}
```


## $type
```
db.persons.aggregate([
    {$project: {
        name: 1,
        nameType: {$type: "$name"},
        ageType: {$type: "$age"},
        tagsType: {$type: "$tags"},
        companyType: {$type: "$company"}
     }}
])
```

```
/* 1 */
{
    "_id" : ObjectId("62416ea2c2705ea08f624389"),
    "name" : "Aurelia Gonzales",
    "nameType" : "string",
    "ageType" : "int",
    "tagsType" : "array",
    "companyType" : "object"
}

/* 2 */
{
    "_id" : ObjectId("62416ea2c2705ea08f62438a"),
    "name" : "Kitty Snow",
    "nameType" : "string",
    "ageType" : "int",
    "tagsType" : "array",
    "companyType" : "object"
}

...
```


## $out
Save results to other collection. Must be the last stage in the pipeline.  If output collection doesn't exist, it will be created automatically.

```
db.persons.aggregate([
    {$project: {
        name: 1,
        nameType: {$type: "$name"},
        ageType: {$type: "$age"},
        tagsType: {$type: "$tags"},
        companyType: {$type: "$company"}
     }},
     {$out: "outCollection"}
])
```


## allowDiskUse: true
Use temprorary files instead of RAM. Need if you need to aggregate large data.

All aggregation stages can use maximum 100MB of RAM. Server will return error if RAM limit is exceeded.

```
db.persons.aggregate([], {allowDiskUse: true})
```


## some helpful links
- MongoDB Aggregation Framework: https://www.youtube.com/watch?v=A3jvoE0jGdE&list=PLWkguCWKqN9OwcbdYm4nUIXnA2IoXX0LI
- Building a CRUD App with FastAPI and MongoDB https://testdriven.io/blog/fastapi-mongo/
- Trees structure with parent: https://www.mongodb.com/docs/manual/tutorial/model-tree-structures-with-parent-references/
- recursive search on a collection: https://www.mongodb.com/docs/manual/reference/operator/aggregation/graphLookup/#mongodb-pipeline-pipe.-graphLookup
- Mongodb trees use cases: https://vdocuments.mx/webinar-working-with-graph-data-in-mongodb.html?page=48
