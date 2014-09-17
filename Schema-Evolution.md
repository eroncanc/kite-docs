---
layout: page
title: Schema Evolution
---

Over time, you might want to add or remove fields in an existing schema. The precise rules for schema evolution are inherited from Avro, and are documented in the Avro specification as rules for [Avro schema resolution][]. For the purposes of working in Kite, here are some important things to note.

## Writer Schemas and Reader Schemas

Writer schemas describe a dataset as it is being written. Reader schemas describe a dataset as it is being read from a datastore. Writer and reader schemas must be compatible, but they do not have to match exactly. See the [Avro schema resolution][] specification for the exhaustive list of rules for matching one schema to another.

## Changing Field Types

You can use schema resolution to change the type used to store a value. For example, you can change an `int` to a `long` to handle values that grow larger than initially anticipated.

## Removing Fields from a Dataset

When you remove fields from a dataset schema, the data already written remains unchanged. The fields you remove are not required when records are written going forward. The field must not be added back, unless it is identical to the existing field (since the data isn't actually removed from the dataset).

## Adding Fields to a Dataset

You can add fields to a dataset's schema, provided the the schema is compatible with the existing data. If you do so, you must define a default value for the fields you add to the dataset schema. New data that includes the field will be populated normally. Records that do not include the field are populated with the default you provide.

## Reading with Different Schemas

You can have a schema that reads fewer fields than are defined by the schema used to write a dataset, provided that the field definitions in the reader schema are compatible with the chosen fields in the writer schema. This is useful when the writer schema provides more fields than are needed for the business case supported by the reader schema.

Removing unnecessary fields allows Kite to read data more efficiently. The performance gain can be significant when using Parquet format, in particular.

Kite ensures that each change to a schema is compatible with the last version of the schema. Older data can always be read by the current schema.

## Example of Schema Evolution

Here's an example that demonstrates how to use the Kite CLI to update the schema for a movies dataset. Complete sample data is available in [movies.tar.gz](../samples/movies.tar.gz).

Begin with a CSV data file (movies.csv).

```
id,title
1,Anticoagulance
2,"Boy and His Cattle, A"
3,Carpool to Vermont
```

Generate an Avro schema file (`movies.avsc`) using `movies.csv`.

```
$ dataset csv-schema movies.csv --class movies -o movies.avsc
```

The schema `movies.avsc` describes fields for _id_ number and the _title_ of the movie.

```
{
  "type" : "record",
  "name" : "movie",
  "doc" : "Schema generated by Kite",
  "fields" : [ {
    "name" : "id",
    "type" : [ "null", "long" ],
    "doc" : "Type inferred from \"1\""
  }, {
    "name" : "title",
    "type" : [ "null", "string" ],
    "doc" : "Type inferred from \"Anticoagulance\""
  } ]
}
```


Create the _movies_ dataset.

```
$ dataset create movies --schema movies.avsc
```

Import the CSV data.

```
$ dataset csv-import movies.csv movies
```

Validate the dataset by showing the first few records.

```
$ dataset show movies
{"id": 1, "title": "Anticoagulance"}
{"id": 2, "title": "Boy and His Cattle, A"}
{"id": 3, "title": "Carpool to Vermont"}
```

Now that you've created your dataset, you immediately receive a request from your director to add a field for a movie rating, 1 through 5. Sigh. You modify the Avro schema file to add the _rating_ field. When you add a new field, you must provide a default value that is used to fill in the field for existing records.

In this case, the default value is _null_. Note that you don't put quotation marks around _null_ when setting it as the default value.

The source code for this file is `movies2.avsc`.

```
{
  "type" : "record",
  "name" : "movies",
  "doc" : "Schema generated by Kite",
  "fields" : [ {
    "name" : "id",
    "type" : [ "null", "long" ],
    "doc" : "Type inferred from '1'"
  }, {
    "name" : "title",
    "type" : [ "null", "string" ],
    "doc" : "Type inferred from 'Anticoagulance'"
  }, {
    "name" : "rating",
    "type" : [ "null", "long"],
    "default" : null,
    "doc" : "Movie rating."
  } ]
}
```

Use the CLI `update` command to add the new field.

```
$ dataset update movies --schema movies2.avsc
```

Now you can load more records that include values for the _rating_ field.  These records are in the file `movies2.csv`.
 
```
id,title,rating
4,Chameleon Chameleon,5
5,Champion of Mediocrity,3
6,Chest Pains,1

```

After you import `movies2.csv`, the existing records display _null_ for the _rating_ field.

```
$ dataset csv-import movies2.csv movies
Added 3 records to "movies"
$ dataset show movies
{"id": 1, "title": ""Anticoagulance", "rating": null}
{"id": 2, "title": "Boy and His Cattle, A", "rating": null}
{"id": 3, "title": "Carpool to Vermont", "rating": null}
{"id": 4, "title": "Chameleon Chameleon", "rating": 5}
{"id": 5, "title": "Champion of Mediocrity", "rating": 3}
{"id": 6, "title": "Chest Pains", "rating": 1}
```

What a complete and satisfying movies dataset. But not so fast. Your director realizes that the _rating_ field should actually allow decimals to store the average ratings from multiple reviewers.

Update the schema definition, changing the _rating_ field datatype from _long_ to _double_. The source code for this file is `movies3.avsc`.

```
{
  "type" : "record",
  "name" : "movies",
  "doc" : "Schema generated by Kite",
  "fields" : [ {
    "name" : "id",
    "type" : [ "null", "long" ],
    "doc" : "Type inferred from '1'"
  }, {
    "name" : "title",
    "type" : [ "null", "string" ],
    "doc" : "Type inferred from 'Anticoagulance'"
  }, {
    "name" : "rating",
    "type" : [ "null", "double" ],
    "default" : null,
    "doc" : "Movie rating."
  } ]
}
```

Once again, update the schema.

```
$ dataset update movies --schema movies3.avsc
```
If you run the `show` command, you'll see that the existing integer _id_ field values now display values with a decimal point and 0.

```
$ dataset show movies
{"id": 1, "title": ""Anticoagulance", "rating": null}
{"id": 2, "title": "Boy and His Cattle, A", "rating": null}
{"id": 3, "title": "Carpool to Vermont", "rating": null}
{"id": 4, "title": "Chameleon Chameleon", "rating": 5.0}
{"id": 5, "title": "Champion of Mediocrity", "rating": 3.0}
{"id": 6, "title": "Chest Pains", "rating": 1.0}

```
The _rating_ values are small, and could easily fit into a _float_ datatype. However, the current datatype is _long_. You have to convert the field to a _double_ datatype, because the highest potential value in a _long_ integer is too high to store in a _float_ field.

The datafile `movies3.csv` contains records with decimal _rating_ numbers.

```
id,title,rating
7,Chuck Maggot,2.5
8,"Cock Crows Thrice, The",3.2
9,Cormack!,4.75
```


Import the records and show the table.

```
$ dataset csv-import movies3.csv movies
Added 3 records to "movies"
$ dataset show movies
{"id": 1, "title": ""Anticoagulance", "rating": null}
{"id": 2, "title": "Boy and His Cattle, A", "rating": null}
{"id": 3, "title": "Carpool to Vermont", "rating": null}
{"id": 4, "title": "Chameleon Chameleon", "rating": 5.0}
{"id": 5, "title": "Champion of Mediocrity", "rating": 3.0}
{"id": 6, "title": "Chest Pains", "rating": 1.0}
{"id": 7, "title": "Chuck Maggot", "rating": 2.5}
{"id": 8, "title": "Cock Crows Thrice, The", "rating": 3.2}
{"id": 9, "title": "Cormack!", "rating": 4.75}
```

See [Avro schema resolution][] for further options.

[Avro Schema resolution]: http://avro.apache.org/docs/current/spec.html#Schema+Resolution "schemaSpec"