---
layout: post
title:  "PostgreSQL, JSON and Hibernate	"
date:   2019-09-13
categories: PostgreSQL postgres JSON jsonb hibernate JPA
---

#PostgreSQL, JSON and Hibernate	

If you ever worked on a project which use SQL database, there is a possibility that in one point of time you've added a text column where you stored arbitary JSON data.
This may be usefull when you have lot of metadata to store which is not used by system and is not very important. 

Example of this would be having users database where user is described as:

```sql
CREATE TABLE users (
   user_id serial PRIMARY KEY,
   username VARCHAR (50) UNIQUE NOT NULL,
   password VARCHAR (50) NOT NULL,
   email VARCHAR (355) UNIQUE NOT NULL
);
```

Now we give user ability to describe himself and each part of his body, like: eye color, hair color, hair length, every bone in body...
You also need to store all kind of structured data, for example entire familiy tree.

A logical approaches would be to maintain additional tables with having 100s of columns or we could simply store this data as JSON within user table.

The problem starts when somebody starts questioning this data, how we can use it, how we can filter it etc...

Well, you regret you didn't go with additional tables and structuring this data in SQL manner? No worries, PostgreSQL team already though about this and implemented a special data type called [JSON or JSONB](https://www.postgresql.org/docs/9.4/datatype-json.html).

## JSON & JSONB

PostgreSQL introduced JSON types with version 9.4 allowing us to store arbitary json data into a single column and be able to query this data. 

>There are two JSON data types: json and jsonb. They accept almost identical sets of values as input. The major practical difference is one of efficiency. The json data type stores an exact copy of the input text, which processing functions must reparse on each execution; while jsonb data is stored in a decomposed binary format that makes it slightly slower to input due to added conversion overhead, but significantly faster to process, since no reparsing is needed. jsonb also supports indexing, which can be a significant advantage.

So, in order to create json column, we'll create a new table as:

```sql
CREATE TABLE users (
   user_id serial PRIMARY KEY,
   username VARCHAR (50) UNIQUE NOT NULL,
   password VARCHAR (50) NOT NULL,
   email VARCHAR (355) UNIQUE NOT NULL,
   metadata jsonb
);
```
Under field metadata we'll store arbitary structured json data. For example, one of metadata json could look like:

```json
  {
    "age": 23,
    "name": "Hopkins Miranda",
    "gender": "male",
    "email": "hopkinsmiranda@sureplex.com",
    "phone": "+1 (850) 544-2748",
    "address": "369 Arlington Place, Omar, Wyoming, 1635",
    "about": "Ullamco veniam est officia aute cillum. Nisi do dolor aliquip aute sint. Nulla commodo dolore amet occaecat anim eiusmod elit sit mollit veniam ea reprehenderit. Aliquip ipsum ex veniam esse ut elit duis proident deserunt. Esse proident excepteur deserunt occaecat exercitation ullamco fugiat sit. Deserunt aute enim duis cillum in aliquip fugiat tempor in. Cupidatat pariatur magna velit do nulla magna quis pariatur minim sit culpa commodo aliquip.\r\n",
    "related": [
      {
        "id": 0,
        "name": "Jerry Powers"
      },
      {
        "id": 1,
        "name": "Sara Barron"
      },
      {
        "id": 2,
        "name": "Pearson Griffith"
      }
    ],
    "parents": {
      "father": {
        "id": 0,
        "name": "Garza George"
      },
      "mother": {
        "id": 0,
        "name": "Wright Mcknight"
      }
    }
  }
```

### Querying JSON

In order to query json columns, PostgreSQL supports the following json operators:

| Operator | Right Operand Type | Description                                                                        | Example                                                   | Example Result |
|----------|--------------------|------------------------------------------------------------------------------------|-----------------------------------------------------------|----------------|
| \->      | int                | Get JSON array element \(indexed from zero, negative integers count from the end\) | '\[\{"a":"foo"\},\{"b":"bar"\},\{"c":"baz"\}\]'::json\->2 | \{"c":"baz"\}  |
| \->      | text               | Get JSON object field by key                                                       | '\{"a": \{"b":"foo"\}\}'::json\->'a'                      | \{"b":"foo"\}  |
| \->>     | int                | Get JSON array element as text                                                     | '\[1,2,3\]'::json\->>2                                    | 3              |
| \->>     | text               | Get JSON object field as text                                                      | '\{"a":1,"b":2\}'::json\->>'b'                            | 2              |
| \#>      | text\[\]           | Get JSON object at specified path                                                  | '\{"a": \{"b":\{"c": "foo"\}\}\}'::json\#>'\{a,b\}'       | \{"c": "foo"\} |
| \#>>     | text\[\]           | Get JSON object at specified path as text                                          | '\{"a":\[1,2,3\],"b":\[4,5,6\]\}'::json\#>>'\{a,2\}'      | 3              |

There are also [additional operators](https://www.postgresql.org/docs/current/functions-json.html)  that could be used in case we need them.


So lets start with something simple. Let say we want to fetch `user_id`and `name` fields from our `metadata` column for all users in database. 

```sql
SELECT user_id as id, metadata->'name' as name from users;
``` 

Note that `->` operator returns json data type. If you want this as text, use `->>`:

```sql
SELECT user_id as id, metadata->>'name' as name from users;
``` 

| "id" | "name"              |
|------|---------------------|
| 1    | "Jeri Henson"       |
| 2    | "Glenda Beasley"    |
| 3    | "Virginia Herring"  |
| 4    | "Hale Riddle"       |
| 5    | "John Woodard"      |

Similary, if we want to put condition on one of the json properties, we could do the following:

```sql
SELECT user_id AS id, metadata->>'name' AS name FROM users WHERE metadata->>'age' = '30';
``` 

Note that even though `age` is number in our json data, operator `->>` will return data as text. If we want this as number we'll need to manually convert it.

```sql
SELECT user_id AS id, metadata->>'name' AS name FROM users WHERE CAST(metadata->>'age' AS int) = 30;
```

Now lets query by some nested fields, for example lets query all user where father's name starts with `Alvarez`.

```sql
SELECT user_id AS id, metadata->>'name' AS name FROM users WHERE metadata->'parents'->'father'->>'name' LIKE 'Alvarez%';
```

Notice how we use `->` operator to walkt through json fields where each time we use `->` next object we get is again json. At the end we use `->>` to fetch this data as text.

In this case we could also use simple way of fetching this data using operator `#>` and `#>>`. Similary to `->` and `->>`, `#>` fetches data as json and `#>>` fetches data as text.

```sql
SELECT user_id AS id, metadata->>'name' AS name FROM users WHERE metadata#>>'{parents,father,name}' LIKE 'Alvarez%';
```

To fetch data from arrays, we'll need to specify index in array we want to fetch. For example:

```
SELECT user_id AS id, metadata#>>'{related,0,name}' AS name FROM users

SELECT user_id AS id, metadata->'related'->0->>'name' AS name FROM users; #Note how index is not single quoted
```

### Java, Postgres and Hibernate and `pg-jsonb`

Now when we know how to query json data in postgres, we can use this in our application. Recently I've release [java libary](https://github.com/vzornic/pg-jsonb) for eaiser querying this data with JDBC or Hibernate.
Libary also expose low level API for building these queries so it is easy to addopt for any other ORM.

##### Installing

To install pg-json add the following in your pom.xml:

```
<dependency>
    <groupId>com.vzornic.pgjsonb</groupId>
    <artifactId>pgjsonb</artifactId>
    <version>${version}</version>
</dependency>
```

#### Low Level API

`pg-json` expose low level API which can be used along with native queries or can be incorporeted in other popular java ORMs.

Most important part of API is `JsonProperty` which is used to initialize and later use data selector for json data.

To init JsonProperty use one of the following constructors:

```
JsonProperty(String column, String path)
```
The `column` parameter is name of the json column.
The `path` property is path we're trying to query inside json column. This path is specified with javascript notation. For example, we can init property as:

```
JsonProperty propery = new JsonProperty("metadata", "parents.father.name");
```

The other constructor allows specifing path in more structured way as List of JsonPath. 

```js
JsonProperty(String column, List<JsonPath> paths)
```

Example of this would be:

```
List<JsonPath> path = new ArrayList<>();
path.add(new JsonPath("related", PathType.STRING));
path.add(new JsonPath("0", PathType.NUMBER));
path.add(new JsonPath("name", PathType.STRING));

JsonProperty propery = new JsonProperty("metadata", path);
```

Method `toSqlString()` will output String value of SQL path for the JsonProperty. For example:

```
JsonProperty propery = new JsonProperty("metadata", "parents.father.name");
system.out.println(property.toSqlString());

//"metadata#>>'{parents,father,name}'" 
```

##### Additional Options

- `asJson()`
This will output JsonProperty as json or jsonb data type. By default it is false

- `asString()`
Opposite of `asJson()`. By default it is true

- `ignoreValues()`
This will output sql string replacing json paths with ?. For example:

```
--- "metadata#>>'{?,?,?}'" ----
```
Parameters can be retrieved using `getParametrizedValues()`. This is usefull when json path is arbitary string provided by user to prevent SQL injection.

- `randomKeyValues()`
This should be used along with `ignoreValues()`. Instead using `?` to paramterize json path, this will generate random 5 characters long string as parameter name. For example:

```
--- "metadata#>>'{:gRgGH,:TYAlV,:NrOAD}'" ----
```

Same as with `?`, `getParametrizedValues()` is used to retrieve the paremters and its keys.

- `alias(String alias)`
Define alias for column.

- `cast(CastType castType)`
Defines if value from json should be cast to something.
Currently supported values for cast are:
```
NUMERIC
BOOLEAN
INTEGER
BIGINT
DOUBLE_PRECISION
SMALLINT
```

- `mode(JsonIteratorMode mode)`
You can specify mode/operator query should be using. By default 'hashtag' (`#>` and `#>>`) operator will be used. You can use `ARROW_ITERATOR_MODE` mode to output queries using `->` and `->>` operators.

- `hql()`
This will output query to internal HQL representation. pg-jsonb defines HQL function in order to generate query. An example of this would be:

```
internal_json_text_default('metadata', '{parents,father,name}');
```

`internal_json_text_default` is HQL function which will produce the SQL query
```
--- "metadata#>>'{parents,father,name}'" ----
```

#### Low level API conditions

Low level API also expose set of conditions to build queries. At the time of writing this blog, API supports the following conditions:

- `SimpleJsonCondition` compares json property with a value using user provided operator
- `BetweenJsonCondition` compares json property with values using BETWEEN operator
- `InJsonCondition` compares json property with values using IN operator
- `NullJsonCondition` checks if json property is null.

###### Condition examples:

```javascript
SimpleJsonCondition condition = new SimpleJsonCondition(new JsonProperty("data", "one.two.three[0]"), new ParametrizedValue("value"), "=");
//"data#>>'{one,two,three,0}'=value

BetweenJsonCondition condition = new BetweenJsonCondition(new JsonProperty("data", "one.two.three[0]"), new ParametrizedValue(10), new ParametrizedValue(20));
//data#>>'{one,two,three,0}' BETWEEN 10 AND 20

InJsonCondition condition = new InJsonCondition( new JsonProperty("data", "one.two.three[0]"), Arrays.asList(new ParametrizedValue(10), new ParametrizedValue(20)));
//data#>>'{one,two,three,0}' IN ( 10,20 )

NullJsonCondition condition = new NullJsonCondition(new JsonProperty("data", "one.two.three[0]"), true);
//data#>>'{one,two,three,0}' IS NULL
```

#### Hibernate

Depending on version of Hibernate, we can use pg-jsonb with `CriteriaBuilder` or `Criterion`  API.

##### CriteriaBuilder API

First we'll need to use one of dialects provided with libary. This is required in order to be able to parse HQL queries. Available dialects you can find [here](https://github.com/vzornic/pg-jsonb/tree/master/src/main/java/com/vzornic/pgjson/hibernate/dialect).

In order to query we'll need to get Hibernates implementation of JPAs `Root`. The usual way how we build queries is with hiberante and JPA:

```
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<X> cr = cb.createQuery(X.class);
Root<X> root = cr.from(X.class);
```

Now we need to initialize `JSONRootImpl<X>` using previosly retrieved root.

```
JSONRootImpl<User> jsonRoot = new JSONRootImpl<User>((RootImpl<User>) root, "metadata");
```

Now we can use `jsonRoot` to generate paths for our queries with usual way how we build queries with CriteriaBuilder. 
We'll use `get(String jsonPath)` to query string value of json path or `get(String jsonPath, Class<Y> type)` to query `Y type` value of json path. Supported `Y` values are provided in `CastType` low level API.

Examples:

```js
cr.where(cb.equal(jsonRoot.get("parent.father.name"), "John Doe"));
cr.where(cb.gt(jsonRoot.get("parent.father.age", Integer.class), 50));
```

You can find more examples in our [tests](https://github.com/vzornic/pg-jsonb/blob/master/src/test/java/com/vzornic/pgjson/hibernate/expressions/JSONBCriteriaBuilderExpressionTest.java).



##### ~~Criterion API~~ (deprecated with newer versions of hibernate)

In order to json queries with Criterion API we'll be using `JSONBRestrictions` which, similary to hibernates `Restrictions`, provide set of queries to use.

Examples:

```js
Criteria criteria;// Get Criteria 
criteria.add(JSONBRestrictions.eq("metadata", "parent.father.name", "John Doe"); 
criteria.add(JSONBRestrictions.gt("metadata", "parent.father.age", 30);
```

### Additional

It is also possible to extends Hibernate `UserType` to be able to map json data type into more structured Java class. However, you might want to rethink this as often you will want this data to be flexible enough so you don't want to care about maintaining its structure.

`jsonb` data type also supports GIN indexes which could be used to efficiently search over json.

### Conclusion

As great as it is to have this kind of flexibility along with SQL database, don't abuse it. There are use cases for using this field, but just because its cool and gives you flexibility doesn't mean you want to use it.

 Remember that json data types do not have schema/mapping so postgres doesn't know what you'll insert. You'll need to be careful what you will query, especially when it comes to casting to different data types.





