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
- **Mongodb compass**
- Mongodb plugin for vscode. Create new playground in vscode to run.
- Robo 3t

**howto increase font size in Robo 3t?**

In file `/Users/<user>/.3T/robo-3t/1.4.4/robo3t.json` change `"textFontPointSize": -1` -> `"textFontPointSize": 15`


## structure and syntax
Field path: `"$filedName"`
System variable: `"$$UPPERCASE"`
User variable: `"$$foo"`

- Piplenes are always an array of one or more stages
- Stages are composed of one or more aggregation operators or expressions
- Expression may take a single argument or an array of arguments. 


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
![[Screenshot 2022-03-29 at 15.19.28 1.png]]

- A $match stage may contain a $text query operator, but it must be the first stage in the pipeline.
- $match come early in an aggregation pipeline
- you cannot use $where with match
- $match uses the same query syntax as find


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
1 ascending
-1 descending

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
- once we specify one field to retain, we must specify all fields we want to retain, `_id` feild is exception
- $project can be used as manyt times as required
- can be used to derive entirely new fileds

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
- only work on array values
- two froms short and long

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

## $lookup two collections
Leverages the left outer join method to merge information from one document to another. You could add an extra field to the existing document to demystify information

[![Quantity diagrams: INNER JOIN vs. OUTER JOIN](https://www.ionos.com/digitalguide/fileadmin/DigitalGuide/Screenshots_2018/Outer-Join.jpg "Quantity diagrams: INNER JOIN vs. OUTER JOIN")](https://www.ionos.com/digitalguide/fileadmin/DigitalGuide/Screenshots_2018/Outer-Join.jpg)

Two collections:
```
db.orders.insertMany( [
   { "_id" : 1, "item" : "almonds", "price" : 12, "quantity" : 2 },
   { "_id" : 2, "item" : "pecans", "price" : 20, "quantity" : 1 },
   { "_id" : 3  }
] )

db.inventory.insertMany( [
   { "_id" : 1, "sku" : "almonds", "description": "product 1", "instock" : 120 },
   { "_id" : 2, "sku" : "bread", "description": "product 2", "instock" : 80 },
   { "_id" : 3, "sku" : "cashews", "description": "product 3", "instock" : 60 },
   { "_id" : 4, "sku" : "pecans", "description": "product 4", "instock" : 70 },
   { "_id" : 5, "sku": null, "description": "Incomplete" },
   { "_id" : 6 }
] )
```

Example 1
```
db.orders.aggregate( [
   {
     $lookup:
       {
         from: "inventory",
         localField: "item",
         foreignField: "sku",
         as: "inventory_docs"
       }
  }
] )
```

```
{
   "_id" : 1,
   "item" : "almonds",
   "price" : 12,
   "quantity" : 2,
   "inventory_docs" : [
      { "_id" : 1, "sku" : "almonds", "description" : "product 1", "instock" : 120 }
   ]
}
{
   "_id" : 2,
   "item" : "pecans",
   "price" : 20,
   "quantity" : 1,
   "inventory_docs" : [
      { "_id" : 4, "sku" : "pecans", "description" : "product 4", "instock" : 70 }
   ]
}
{
   "_id" : 3,
   "inventory_docs" : [
      { "_id" : 5, "sku" : null, "description" : "Incomplete" },
      { "_id" : 6 }
   ]
}
```


## $lookup -> $mergeObjects
```
db.orders.aggregate( [
   {$lookup:
       {
         from: "inventory",
         localField: "item",
         foreignField: "sku",
         as: "inventory_docs"
       }
  },
  {
      $replaceRoot: {
          newRoot: {
              $mergeObjects: [ { $arrayElemAt: [ "$inventory_docs", 0 ] }, "$$ROOT" ] } 
      }
   },
   {$project: {inventory_docs: 0}}
] )
```

```

/* 1 */
{
    "_id" : 1.0,
    "sku" : "almonds",
    "description" : "product 1",
    "instock" : 120.0,
    "item" : "almonds",
    "price" : 12.0,
    "quantity" : 2.0
}

/* 2 */
{
    "_id" : 2.0,
    "sku" : "pecans",
    "description" : "product 4",
    "instock" : 70.0,
    "item" : "pecans",
    "price" : 20.0,
    "quantity" : 1.0
}

/* 3 */
{
    "_id" : 3.0,
    "sku" : null,
    "description" : "Incomplete"
}
```


## $lookup one collection
```
db.collection_1.aggregate( [
    {$lookup:
       {
         from: "collection_1",
         localField: "_id",
         foreignField: "parent_id",
         as: "children"
       }
    },
    {$project: {
        _id: 1,
        parent_id: 1,
        children: "$children._id"
        }
    }
  
])
```
Result the same as $graphLookup with maxDepth=0
```
/* 1 */
{
    "_id" : 1.0,
    "parent_id" : null,
    "children" : [ 
        2.0, 
        3.0
    ]
}

/* 2 */
{
    "_id" : 2.0,
    "parent_id" : 1.0,
    "children" : []
}

/* 3 */
{
    "_id" : 3.0,
    "parent_id" : 1.0,
    "children" : [ 
        4.0, 
        5.0
    ]
}

/* 4 */
{
    "_id" : 4.0,
    "parent_id" : 3.0,
    "children" : []
}

/* 5 */
{
    "_id" : 5.0,
    "parent_id" : 3.0,
    "children" : [ 
        6.0
    ]
}

/* 6 */
{
    "_id" : 6.0,
    "parent_id" : 5.0,
    "children" : []
}
```


## $graphLookup

Prepare collection
```
db.collection_1.insertMany([
	{ '_id': 1, 'parent_id': null },
	{ '_id': 2, 'parent_id': 1 },
	{ '_id': 3, 'parent_id': 1 },
	{ '_id': 4, 'parent_id': 3 },
	{ '_id': 5, 'parent_id': 3 },
	{ '_id': 6, 'parent_id': 5 }
])
```

Example
```
db.collection_1.aggregate([
    {$graphLookup: {
		from: 'collection_1',
		startWith: "$_id",
		connectFromField: '_id',
		connectToField: 'parent_id',
		as: 'children'
		}
    },
    {$project: {
        _id: 1,
        parent_id: 1,
        children: "$children._id"
        }
    }
])
```

```
/* 1 */
{
    "_id" : 1.0,
    "parent_id" : null,
    "children" : [ 
        3.0, 
        4.0, 
        6.0, 
        2.0, 
        5.0
    ]
}

/* 2 */
{
    "_id" : 2.0,
    "parent_id" : 1.0,
    "children" : []
}

/* 3 */
{
    "_id" : 3.0,
    "parent_id" : 1.0,
    "children" : [ 
        4.0, 
        6.0, 
        5.0
    ]
}

/* 4 */
{
    "_id" : 4.0,
    "parent_id" : 3.0,
    "children" : []
}

/* 5 */
{
    "_id" : 5.0,
    "parent_id" : 3.0,
    "children" : [ 
        6.0
    ]
}

/* 6 */
{
    "_id" : 6.0,
    "parent_id" : 5.0,
    "children" : []
}

```


## $graphLookup with maxDepth
```
db.collection_1.aggregate([
    {$graphLookup: {
        from: 'collection_1',
        startWith: "$_id",
        connectFromField: '_id',
        connectToField: 'parent_id',
        maxDepth: 0,
        as: 'children'
        }
    },
    {$project: {
        _id: 1,
        parent_id: 1,
        children: "$children._id"
        }
    }
])
```

```
/* 1 */
{
    "_id" : 1.0,
    "parent_id" : null,
    "children" : [ 
        3.0, 
        2.0
    ]
}

/* 2 */
{
    "_id" : 2.0,
    "parent_id" : 1.0,
    "children" : []
}

/* 3 */
{
    "_id" : 3.0,
    "parent_id" : 1.0,
    "children" : [ 
        5.0, 
        4.0
    ]
}

/* 4 */
{
    "_id" : 4.0,
    "parent_id" : 3.0,
    "children" : []
}

/* 5 */
{
    "_id" : 5.0,
    "parent_id" : 3.0,
    "children" : [ 
        6.0
    ]
}

/* 6 */
{
    "_id" : 6.0,
    "parent_id" : 5.0,
    "children" : []
}
```


## $geoNear
- collection have one and only one 2sphere index
- if using 2dsphere, the distance return in meters
- $geoNear must be the first stage


## $sample
`{$sample: {size: < N, how many documents}}`

- N <= 5% if documents in collection AND
- source collection >= 100 documents AND
- $sample is the first stage

## $redact
- can be useful to implementing access controll list
- `$$KEEP` and `$$PRUNE` automatically apply to all levels below the evaluated level
- `$$DESCEND` retains the current level and evaluates the next level down
- $redact isnot for restricting access to a collection

## MongoDB University, M121: The MongoDB Aggregation Framework


### Lab - $match
-   imdb.rating is at least 7
-   genres does not contain "Crime" or "Horror"
-   rated is either "PG" or "G"
-   languages contains "English" and "Japanese"

```
{ $match:
	{
		"imdb.rating": { $gte: 7 },
		genres: {$nin: ["Crime", "Horror"]},
		$or: [{rated: "PG"}, {rated: "G"}],
		$and: [{languages: "English"}, {languages: "Japanese"}]
	}

}
```


### Lab - Changing Document Shape with $project
Using the same $match stage from the previous lab, add a $project stage to only display the title and film rating (title and rated fields)

```
var pipeline = [ 
	{$match:
		{
			"imdb.rating": { $gte: 7 },
			genres: {$nin: ["Crime", "Horror"]},
			$or: [{rated: "PG"}, {rated: "G"}],
			$and: [{languages: "English"}, {languages: "Japanese"}]
		}

	},
	{$project:
		{
			_id: 0,
			title: 1,
			rated: 1
		}	
	}
]
```


### Lab - Computing Fields
Using the Aggregation Framework, find a count of the number of movies that have a title composed of one word. To clarify, "Cinderella" and "3-25" should count, where as "Cast Away" would not.

```
db.movies.aggregate([ 
	{$project:
		{
			title: 1,
			words: {$size: {$split: ["$title", " "]}}
		}			
	},
	{$match:
		{
			words: {$eq: 1}
		}
	},
	{$count: "oneWordTitles"}
])
```


### Optional Lab - Expressions with $project
Let's find how many movies in our **movies** collection are a "labor of love", where the same person appears in cast, directors, and writers

```
[{$match: {
 writers: {
  $elemMatch: {
   $exists: true
  }
 },
 cast: {
  $elemMatch: {
   $exists: true
  }
 },
 directors: {
  $elemMatch: {
   $exists: true
  }
 }
}}, {$project: {
 writers: {
  $map: {
   input: '$writers',
   as: 'writer',
   'in': {
    $arrayElemAt: [
     {
      $split: [
       '$$writer',
       ' ('
      ]
     },
     0
    ]
   }
  }
 },
 cast: 1,
 directors: 1
}}, {$project: {
 inAll: {
  $gt: [
   {
    $size: {
     $setIntersection: [
      '$writers',
      '$cast',
      '$directors'
     ]
    }
   },
   0
  ]
 }
}}, {$match: {
 inAll: true
}}, {$count: 'all'}]
```


### Lab: Using Cursor-like Stages
For movies released in the **USA** with a tomatoes.viewer.rating greater than or equal to 3, calculate a new field called num_favs that represets how many **favorites** appear in the cast field of the movie.

Sort your results by num_favs, tomatoes.viewer.rating, and title, all in descending order.

What is the title of the **25th** film in the aggregation result?
```
[{$match: {
 countries: 'USA',
 'tomatoes.viewer.rating': {
  $gte: 3
 }
}}, {$addFields: {
 favorites: [
  'Sandra Bullock',
  'Tom Hanks',
  'Julia Roberts',
  'Kevin Spacey',
  'George Clooney'
 ]
}}, {$addFields: {
 favs: {
  $setIntersection: [
   '$cast',
   '$favorites'
  ]
 }
}}, {$addFields: {
 num_favs: {
  $cond: {
   'if': {
    $isArray: '$favs'
   },
   then: {
    $size: '$favs'
   },
   'else': null
  }
 }
}}, {$sort: {
 num_favs: -1,
 'tomatoes.viewer.rating': -1,
 title: -1
}}, {$skip: 24}, {$limit: 1}]
```


### Lab - Bringing it all together
Calculate an average rating for each movie in our collection where English is an available language, the minimum imdb.rating is at least 1, the minimum imdb.votes is at least 1, and it was released in **1990** or after. You'll be required to [rescale (or _normalize_)](https://en.wikipedia.org/wiki/Feature_scaling) imdb.votes. The formula to rescale imdb.votes and calculate normalized_rating is included as a handout.

What film has the lowest normalized_rating?
```
[{$match: {
 languages: 'English',
 'imdb.rating': {
  $gte: 1
 },
 'imdb.votes': {
  $gte: 1
 },
 year: {
  $gte: 1990
 }
}}, {$addFields: {
 scaled_votes: {
  $add: [
   1,
   {
    $multiply: [
     9,
     {
      $divide: [
       {
        $subtract: [
         '$imdb.votes',
         5
        ]
       },
       {
        $subtract: [
         1521105,
         5
        ]
       }
      ]
     }
    ]
   }
  ]
 }
}}, {$project: {
 title: 1,
 normalized_rating: {
  $avg: [
   '$imdb.rating',
   '$scaled_votes'
  ]
 }
}}, {$sort: {
 normalized_rating: 1
}}, {$limit: 1}]
```


### Lab - $group and Accumulators
For all films that won at least 1 Oscar, calculate the standard deviation, highest, lowest, and average imdb.rating. Use the **sample** standard deviation expression.

```
[{$match: {
 awards: {
  $regex: RegExp('Won \d{1,2} Oscar')
 }
}}, {$project: {
 ratign: '$imdb.rating'
}}, {$group: {
 _id: null,
 max: {
  $max: '$ratign'
 },
 min: {
  $min: '$ratign'
 },
 avg: {
  $avg: '$ratign'
 },
 std: {
  $stdDevSamp: '$ratign'
 }
}}]
```


### Lab - $unwind
What is the name, number of movies, and average rating (truncated to one decimal) for the cast member that has been in the most number of movies with **English** as an available language?

```
[{$match: {
 languages: 'English'
}}, {$unwind: {
 path: '$cast'
}}, {$group: {
 _id: '$cast',
 films: {
  $addToSet: '$title'
 },
 average: {
  $avg: '$imdb.rating'
 }
}}, {$project: {
 _id: 1,
 numFilms: {
  $size: '$films'
 },
 average: 1
}}, {$sort: {
 numFilms: -1,
 average: -1
}}]
```


### Lab - Using $lookup
Which alliance from air_alliances flies the most **routes** with either a Boeing 747 or an Airbus A380 (abbreviated 747 and 380 in air_routes)?


### Lab: $graphLookup
Find the list of all possible distinct destinations, with at most one layover, departing from the base airports of airlines from Germany, Spain or Canada that are part of the "OneWorld" alliance. Include both the destination and which airline services that location. As a small hint, you should find **158** destinations.

```
[{
  $match: { name: "OneWorld" }
}, {
  $graphLookup: {
    startWith: "$airlines",
    from: "air_airlines",
    connectFromField: "name",
    connectToField: "name",
    as: "airlines",
    maxDepth: 0,
    restrictSearchWithMatch: {
      country: { $in: ["Germany", "Spain", "Canada"] }
    }
  }
}, {
  $graphLookup: {
    startWith: "$airlines.base",
    from: "air_routes",
    connectFromField: "dst_airport",
    connectToField: "src_airport",
    as: "connections",
    maxDepth: 1
  }
}, {
  $project: {
    validAirlines: "$airlines.name",
    "connections.dst_airport": 1,
    "connections.airline.name": 1
  }
},
{ $unwind: "$connections" },
{
  $project: {
    isValid: { $in: ["$connections.airline.name", "$validAirlines"] },
    "connections.dst_airport": 1
  }
},
{ $match: { isValid: true } },
{ $group: { _id: "$connections.dst_airport" } }
]
```


### Lab - $facets
How many movies are in both the top ten highest rated movies according to the imdb.rating and the metacritic fields? We should get these results with exactly one access to the database.

```
[
  {
    $match: {
      metacritic: { $gte: 0 },
      "imdb.rating": { $gte: 0 }
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
      top_metacritic: [
        {
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
      top_imdb: [
        {
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
]
```

## some helpful links
- MongoDB University, M121: The MongoDB Aggregation Framework https://university.mongodb.com/mercury/M121/2022_March_22/overview
- Aggregation Pipeline Quick Reference https://www.mongodb.com/docs/manual/meta/aggregation-quick-reference/#string-expressions
- MongoDB Aggregation Framework: https://www.youtube.com/watch?v=A3jvoE0jGdE&list=PLWkguCWKqN9OwcbdYm4nUIXnA2IoXX0LI
- Building a CRUD App with FastAPI and MongoDB https://testdriven.io/blog/fastapi-mongo/
- Trees structure with parent: https://www.mongodb.com/docs/manual/tutorial/model-tree-structures-with-parent-references/
- recursive search on a collection: https://www.mongodb.com/docs/manual/reference/operator/aggregation/graphLookup/#mongodb-pipeline-pipe.-graphLookup
- Mongodb trees use cases: https://vdocuments.mx/webinar-working-with-graph-data-in-mongodb.html?page=48
