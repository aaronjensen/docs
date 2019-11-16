# Server Functions

The message store provides an interface of Postgres server functions that can be used with any programming language or through the `psql` command line tool.

There are working examples of uses of the server functions included with the source code:

Example: [https://github.com/eventide-project/postgres-message-store/blob/master/database](https://github.com/eventide-project/postgres-message-store/blob/master/database)

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
SELECT write_message('a11e9022-e741-4450-bf9c-c4cc5ddb6ea3', 'someStream-123', 'SomeMessageType', '{"messageAttribute": "some value"}', '{"metaDataAttribute": "some meta data value"}');
```

Example: [https://github.com/eventide-project/postgres-message-store/blob/master/database/write-test-message.sh](https://github.com/eventide-project/postgres-message-store/blob/master/database/write-test-message.sh)

### Specifying the Expected Version of the Stream

``` sql
SELECT write_message('a11e9022-e741-4450-bf9c-c4cc5ddb6ea3', 'someStream-123', 'SomeMessageType', '{"messageAttribute": "some value"}', '{"metaDataAttribute": "some meta data value"}', 11);
```

NOTE: If the expected version does not match the stream version at the time of the write, an error is raised of the form:

```
'Wrong expected version: {specified_stream_version} (Stream: {stream_name}, Stream Version: {current_stream_version})'
```

Example (_no expected version error_): [https://github.com/eventide-project/postgres-message-store/blob/master/test/write-message-expected-version.sh](https://github.com/eventide-project/postgres-message-store/blob/master/test/write-message-expected-version.sh)

Example (_with expected version error_): [https://github.com/eventide-project/postgres-message-store/blob/master/test/write-message-expected-version-error.sh](https://github.com/eventide-project/postgres-message-store/blob/master/test/write-message-expected-version-error.sh)

## Get Messages from a Stream

Retrieve messages from a single stream, optionally specifying the starting position, the number of messages to retrieve, and an additional condition that will be appended to the SQL command's WHERE clause.

``` sql
get_stream_messages(
  stream_name varchar,
  position bigint DEFAULT 0,
  batch_size bigint DEFAULT 1000,
  correlation varchar DEFAULT NULL,
  condition varchar DEFAULT NULL
)
```

### Arguments

| Name | Type | Description | Default | Example |
| --- | --- | --- | --- | --- |
| stream_name | varchar | Name of stream to retrieve messages from | (none) | someStream-123 |
| position (optional) | bigint | Starting position of the messages to retrieve | 0 | 11 |
| batch_size (optional) | bigint | Number of messages to retrieve | 1000 | 111 |
| correlation (optional) | varchar | Category or stream name recorded in message metadata's `correlationStreamName` attribute to filter the batch by | NULL | someCategory |
| condition (optional) | varchar | SQL condition to filter the batch by | NULL | messages.time >= current_time |

### Usage

``` sql
SELECT * FROM get_stream_messages('someStream-123', 0, 1000, correlation => 'someCateogry', condition => 'messages.time >= current_time');
```

Example: [https://github.com/eventide-project/postgres-message-store/blob/master/test/get-stream-messages.sh](https://github.com/eventide-project/postgres-message-store/blob/master/test/get-stream-messages.sh)

## Get Messages from a Category

Retrieve messages from a category of streams, optionally specifying the starting position, the number of messages to retrieve, the correlation category for Pub/Sub, consumer group parameters, and an additional condition that will be appended to the SQL command's WHERE clause.

``` sql
CREATE OR REPLACE FUNCTION get_category_messages(
  category_name varchar,
  position bigint DEFAULT 0,
  batch_size bigint DEFAULT 1000,
  correlation varchar DEFAULT NULL,
  consumer_group_member varchar DEFAULT NULL,
  consumer_group_size varchar DEFAULT NULL,
  condition varchar DEFAULT NULL
)
```

### Arguments

| Name | Type | Description | Default | Example |
| --- | --- | --- | --- | --- |
| category_name | varchar | Name of the category to retrieve messages from | (none) | someStream |
| position (optional) | bigint | Global position to start retrieving messages from | 1 | 11 |
| batch_size (optional) | bigint | Number of messages to retrieve | 1000 | 111 |
| correlation (optional) | varchar | Category or stream name recorded in message metadata's `correlationStreamName` attribute to filter the batch by | NULL | someCategory |
| consumer_group_member (optional) | bigint | The zero-based member number of an individual consumer that is participating in a consumer group | NULL | 1 |
| consumer_group_size (optional) | bigint | The size of a group of consumers that are cooperatively processing a single input stream | NULL | 2 |
| condition (optional) | varchar | SQL condition to filter the batch by | NULL | messages.time >= current_time |

### Usage

``` sql
SELECT * FROM get_category_messages('someStream', 1, 1000, correlation => 'someCateogry', consumer_group_member => 1, consumer_group_size => 2, condition => 'messages.time >= current_time');
```

::: tip
Where `someStream-123` is a _stream name_, `someStream` is a _category_. Reading the `someStream` category retrieves messages from all streams whose names start with `someStream-`.
:::

Example: [https://github.com/eventide-project/postgres-message-store/blob/master/test/get-category-messages.sh](https://github.com/eventide-project/postgres-message-store/blob/master/test/get-category-messages.sh)

### Pub/Sub and Retrieving Correlated Messages

The principle use of the `correlation` parameter is to implement Pub/Sub.

The `correlation` parameter filters the retrieved batch based on the content of message metadata's `correlation_stream_name` attribute. The correlation stream name is like a _return address_. It's a way to give the message some information about the component where the message originated from. This information is carried from message to message in a workflow until it ultimately returns to the originating component.

The `correlation_stream_name` attribute allows a component to tag an outbound message with its origin. And then later, the originating component can subscribe to other components' events that carry the origin metadata.

Before the source component sends the message to the receiving component, the source component assigns it's own stream name to the message metadata's `correlation_stream_name` attribute. That attribute is carried from message to message through messaging workflows.

For more details on pub/sub using the correlation stream, see the [pub/sub topic in the consumers user guide](../consumers.html#correlation-and-pub-sub).

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
| stream_name | varchar | Name of the stream to retrieve messages from | (none) |  someStream-123 |

### Usage

``` sql
SELECT * FROM get_last_message('someStream');
```

Note: This is only for entity streams, and does not work for categories.

Example: [https://github.com/eventide-project/postgres-message-store/blob/master/test/get-last-message.sh](https://github.com/eventide-project/postgres-message-store/blob/master/test/get-last-message.sh)

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

## Debugging Output

The message store's server functions will print parameter values, and any generated SQL code, to the standard I/O of the client process.

Debugging output can be enabled for all server functions, or for the get functions and the write function separately

### `message_store.debug_get`

The `debug_get` setting controls debug output for the retrieval functions, including `get_stream_messages`, `get_category_messages`, and `get_last_message`.

Assign the value `on` to the setting to enable debug output.

`message_store.debug_get=on`

### `message_store.debug_write`

The `debug_write` setting controls debug output for the write function, `write_message`.

Assign the value `on` to the setting to enable debug output.

`message_store.debug_write=on`

### `message_store.debug`

The `debug` setting controls debug output for the get functions and the write function.

Assign the value `on` to the setting to enable debug output.

`message_store.debug=on`

### Enabling Debug Output Using a Postgres Environment Variable

The debugging output configuration settings can be enabled in a terminal session using the `PGOPTIONS` environment variable.

``` bash
PGOPTIONS="-c message_store.debug=on"
```

### Enabling Debug Output Using the Postgres Configuration File

The debugging output configuration settings can be set using PostgresSQL's configuration file.

The file system location of the configuration file can be displayed at the command line using the `psql` tool.

``` bash
psql -c 'show config_file'
```

### More on Postgres Configuration

See the PostgreSQL documentation for more configuration options:<br />
[https://www.postgresql.org/docs/current/config-setting.html](https://www.postgresql.org/docs/current/config-setting.html)
