# Server Functions

The message store provides an interface of Postgres server functions that you can access with any programming language, or the `psql` command line tool.

There are working examples of uses of the server functions included with the source code:

Example: [https://github.com/eventide-project/message-store-postgres-database/blob/master/database](https://github.com/eventide-project/message-store-postgres-database/blob/master/database)

## Write a Message

Write a JSON-formatted message to a named stream, optionally specifying JSON-formatted metadata and an expected version number.

``` sql
write_message(
  id varchar,
  stream_name varchar,
  type varchar,
  data jsonb,
  metadata jsonb DEFAULT NULL,
  expected_version bigint DEFAULT NULL
)
```

### Arguments

| Name | Type | Description | Default | Example |
| --- | --- | --- | --- | --- |
| id | varchar | UUID of the message being written | | a5eb2a97-84d9-4ccf-8a56-7160338b11e2 |
| stream_name | varchar | Name of stream to which the message is written | | someStream-123 |
| type | varchar | The type of the message | | Withdrawn |
| data | jsonb | JSON representation of the message body | | {"messageAttribute": "some value"} |
| metadata (optional) | jsonb | JSON representation of the message metadata | NULL | {"metaDataAttribute": "some meta data value"} |
| expected_version (optional) | bigint | Version that the stream is expected to be when the message is written | NULL | 11 |

### Usage

``` sql
SELECT write_message('uuid'::varchar, 'stream_name'::varchar, 'message_type'::varchar, '{"messageAttribute": "some value"}'::jsonb, '{"metaDataAttribute": "some meta data value"}'::jsonb);"
```

Example: [https://github.com/eventide-project/message-store-postgres-database/blob/master/database/write-test-message.sh](https://github.com/eventide-project/message-store-postgres-database/blob/master/database/write-test-message.sh)

### Specifying the Expected Version of the Stream

``` sql
SELECT write_message('uuid'::varchar, 'stream_name'::varchar, 'message_type'::varchar, '{"messageAttribute": "some value"}'::jsonb, '{"metaDataAttribute": "some meta data value"}'::jsonb, expected_version::bigint);"
```

NOTE: If the expected version does not match the stream version at the time of the write, an error is raised of the form:

```
'Wrong expected version: % (Stream: %, Stream Version: %)'
```

Example (_no expected version error_): [https://github.com/eventide-project/message-store-postgres-database/blob/master/test/write-message-expected-version.sh](https://github.com/eventide-project/message-store-postgres-database/blob/master/test/write-message-expected-version.sh)

Example (_with expected version error_): [https://github.com/eventide-project/message-store-postgres-database/blob/master/test/write-message-expected-version-error.sh](https://github.com/eventide-project/message-store-postgres-database/blob/master/test/write-message-expected-version-error.sh)

## Get Messages from a Stream

Retrieve messages from a single stream, optionally specifying the starting position, the number of messages to retrieve, and an additional condition that will be appended to the SQL command's WHERE clause.

``` sql
get_stream_messages(
  stream_name varchar,
  position bigint DEFAULT 0,
  batch_size bigint DEFAULT 1000,
  condition varchar DEFAULT NULL
)
```

### Arguments

| Name | Type | Description | Default | Example |
| --- | --- | --- | --- | --- |
| stream_name | varchar | Name of stream to retrieve messages from | | someStream-123 |
| position (optional) | bigint | Starting position of the messages to retrieve | 0 | 11 |
| batch_size (optional) | bigint | Number of messages to retrieve | 1000 | 111 |
| condition (optional) | varchar | WHERE clause fragment | NULL | messages.time >= current_timestamp |

### Usage

``` sql
SELECT * FROM get_stream_messages('stream_name'::varchar, starting_position::bigint, batch_size::bigint, _condition => 'messages.time >= current_timestamp'::varchar);"
```

Example: [https://github.com/eventide-project/message-store-postgres-database/blob/master/test/get-stream-messages.sh](https://github.com/eventide-project/message-store-postgres-database/blob/master/test/get-stream-messages.sh)

## Get Messages from a Stream Category

Retrieve messages from a category or streams, optionally specifying the starting position, the number of messages to retrieve, and an additional condition that will be appended to the SQL command's WHERE clause.

``` sql
CREATE OR REPLACE FUNCTION get_category_messages(
  category_name varchar,
  position bigint DEFAULT 0,
  batch_size bigint DEFAULT 1000,
  condition varchar DEFAULT NULL
)
```

### Arguments

| Name | Type | Description | Default | Example |
| --- | --- | --- | --- | --- |
| category_name | varchar | Name of the category to retrieve messages from | | someStream |
| position (optional) | bigint | Starting position of the messages to retrieve | 0 | 11 |
| batch_size (optional) | bigint | Number of messages to retrieve | 1000 | 111 |
| condition (optional) | varchar | WHERE clause fragment | NULL | messages.time >= current_timestamp |

### Usage

``` sql
SELECT * FROM get_category_messages('cateogry_name'::varchar, starting_position::bigint, batch_size::bigint, _condition => 'messages.time >= current_timestamp'::varchar);"
```

::: tip
Where `someThing-123` is a _stream name_, `someThing` is a _category_. Reading the `someThing` category retrieves messages from all streams whose names start with `someThing-`.
:::

Example: [https://github.com/eventide-project/message-store-postgres-database/blob/master/test/get-category-messages.sh](https://github.com/eventide-project/message-store-postgres-database/blob/master/test/get-category-messages.sh)

## Get Last Message from a Stream

Retrieve the last message in a stream.

``` sql
get_last_message(
  stream_name varchar
)
```

### Arguments

| Name | Type | Description | Default | Example |
| --- | --- | --- | --- | --- |
| stream_name | varchar | Name of the stream to retrieve messages from | |  someStream-123 |

### Usage

``` sql
SELECT * FROM get_last_message('stream_name'::varchar)
```

Note: This is only for entity streams, and does not work for categories.

Example: [https://github.com/eventide-project/message-store-postgres-database/blob/master/test/get-last-message.sh](https://github.com/eventide-project/message-store-postgres-database/blob/master/test/get-last-message.sh)

## Get Message Store Database Schema Version

Retrieve the four octet version number of the message store database.

``` sql
message_store_version()
```

### Usage

``` sql
SELECT message_store_version();
```

The version number will change when the database schema changes. A database schema change could be a change to the `messages` table structure, changes to Postgres server functions, types, indexes, users, or permissions. The version number follows the [SemVer](https://semver.org/) scheme for the last three numbers in the version (the first number is the product generation, and implies a major version change).