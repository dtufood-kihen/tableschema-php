# Table Schema

[![Travis](https://travis-ci.org/frictionlessdata/tableschema-php.svg?branch=master)](https://travis-ci.org/frictionlessdata/tableschema-php)
[![Coveralls](http://img.shields.io/coveralls/frictionlessdata/tableschema-php.svg?branch=master)](https://coveralls.io/r/frictionlessdata/tableschema-php?branch=master)
[![Scrutinizer-ci](https://scrutinizer-ci.com/g/OriHoch/tableschema-php/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/OriHoch/tableschema-php/)
[![Packagist](https://img.shields.io/packagist/dm/frictionlessdata/tableschema.svg)](https://packagist.org/packages/frictionlessdata/tableschema)
[![SemVer](https://img.shields.io/badge/versions-SemVer-brightgreen.svg)](http://semver.org/)
[![Gitter](https://img.shields.io/gitter/room/frictionlessdata/chat.svg)](https://gitter.im/frictionlessdata/chat)

A utility library for working with [Table Schema](https://specs.frictionlessdata.io/table-schema/) in php.


## Features summary and Usage guide

### Installation

```bash
$ composer require frictionlessdata/tableschema
```

### Schema

Schema class provides helpful methods for working with a table schema and related data.

`use frictionlessdata\tableschema\Schema;`

Schema objects can be constructed using any of the following:

* php object
```php
$schema = new Schema((object)[
    'fields' => [
        (object)[
            'name' => 'id', 'title' => 'Identifier', 'type' => 'integer', 
            'constraints' => (object)[
                "required" => true,
                "minimum" => 1,
                "maximum" => 500
            ]
        ],
        (object)['name' => 'name', 'title' => 'Name', 'type' => 'string'],
    ],
    'primaryKey' => 'id'
]);
```

* string containing json
```php
$schema = new Schema("{
    \"fields\": [
        {\"name\": \"id\"},
        {\"name\": \"height\", \"type\": \"integer\"}
    ]
}");
```

* string containg value supported by [file_get_contents](http://php.net/manual/en/function.file-get-contents.php)
```
$schema = new Schema("https://raw.githubusercontent.com/frictionlessdata/testsuite-extended/ecf1b2504332852cca1351657279901eca6fdbb5/datasets/synthetic/schema.json");
```

The schema is loaded, parsed and validated and will raise exceptions in case of any problems.

access the schema data, which is ensured to conform to the specs.

```
$schema->missingValues(); // [""]
$schema->primaryKey();  // ["id"]
$schema->foreignKeys();  // []
$schema->fields(); // ["id" => IntegerField, "name" => StringField]
$field = $schema->field("id");
$field("id")->format();  // "default"
$field("id")->name();  // "id"
$field("id")->type(); // "integer"
$field("id")->constraints();  // (object)["required"=>true, "minimum"=>1, "maximum"=>500]
$field("id")->enum();  // []
$field("id")->required();  // true
$field("id")->unique();  // false
```

validate function accepts the same arguemnts as the Schema constructor but returns a list of errors instead of raising exceptions
```
// validate functions accepts the same arguments as the Schema constructor
$validationErrors = Schema::validate("http://invalid.schema.json");
foreach ($validationErrors as $validationError) {
    print(validationError->getMessage();
};
```

validate and cast a row of data according to the schema
```
$row = $schema->castRow(["id" => "1", "name" => "First Name"]);
```

will raise exception if row fails validation

it returns the row with all native values

```
$row  // ["id" => 1, "name" => "First Name"];
```

validate the row to get a list of errors

```
$schema->validateRow(["id" => "foobar"]);  // ["id is not numeric", "name is required" .. ]
```

### Table

Table class allows to iterate over data conforming to a table schema


instantiate a Table object based on a data source and a table schema.

```
use frictionlessdata\tableschema\DataSources\CsvDataSource;
use frictionlessdata\tableschema\Schema;
use frictionlessdata\tableschema\Table;

$dataSource = new CsvDataSource("tests/fixtures/data.csv");
$schema = new Schema((object)[
   'fields' => [
       (object)['name' => 'first_name'],
       (object)['name' => 'last_name'],
       (object)['name' => 'order'],
   ]
]);
$table = new Table($dataSource, $schema);
```

iterate over the data, all the values are cast and validated according to the schema
```
foreach ($table as $row) {
    print($row["order"]." ".$row["first_name"]." ".$row["last_name"]."\n");
};
```

validate function will validate the schema and get some sample of the data itself to validate it as well
 
```
Table::validate(new CsvDataSource("http://invalid.data.source/"), $schema);
```

### InferSchema

InferSchema class allows to infer a schema based on a sample of the data

```
use frictionlessdata\tableschema\InferSchema;
use frictionlessdata\tableschema\DataSources\CsvDataSource;
use frictionlessdata\tableschema\Table;

$dataSource = new CsvDataSource("tests/fixtures/data.csv");
$schema = new InferSchema();
$table = new Table($dataSource, $schema);

if (Table::validate($dataSource, $schema) == []) {
    var_dump($schema->fields()); // ["first_name" => StringField, "last_name" => StringField, "order" => IntegerField]
};
```

more control over the infer process

```
foreach ($table as $row) {
    var_dump($row); // row will be in inferred native values
    var_dump($schema->descriptor()); // will contain the inferred schema descriptor
    // the more iterations you make, the more accurate the inferred schema might be
    // once you are satisifed with the schema, lock it
    $rows = $schema->lock();
    // it returns all the rows received until the lock, casted to the final inferred schema
    // you may now continue to iterate over the rest of the rows
};
```

### EditableSchema

EditableSchema extends the Schema object with editing capabilities

```
use frictionlessdata\tableschema\EditableSchema;
use frictionlessdata\tableschema\Fields\FieldsFactory;

$schema = new EditableSchema();
```

edit fields
```
$schema->fields([
    "id" => (object)["type" => "integer"],
    "name" => (object)["type" => "string"],
]);
```

appropriate field object is created according to the given descriptor
```
$schema->field("id");  // IntegerField object
```

add / update or remove fields

```
$schema->field("email", (object)["type" => "string", "format" => "email"]);
$schema->field("name", (object)["type" => "string"]);
$schema->removeField("name");
```

set or update other table schema attributes
```
$schema->primaryKey(["id"]);
```


after every change - schema is validated and will raise Exception in case of validation errors

finally, save the schema to a json file

```
$schema->save("my-schema.json");
```


## Important Notes

- Table schema is in transition to v1 - but many datapackage in the wild are still pre-v1
  - At the moment I am developing this library with support only for v1
  - See [this Gitter discussion](https://gitter.im/frictionlessdata/chat?at=58df75bfad849bcf423e5d80) about this transition


## Contributing

Please read the contribution guidelines: [How to Contribute](CONTRIBUTING.md)
