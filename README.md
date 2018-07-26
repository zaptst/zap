# ZAP ("Ze'test Anything Protocol")

> A streamable structured interface for reporting tests in real time

## Format

The output is stream of events written in newline-delimited json:

```json
{"kind":"suite","event":"started","id":"0","source":"/path/to/test.js:32:4","message":"DatabaseConnection","timestamp":"2018-07-25T23:47:57.133Z"}
{"kind":"test","event":"started","id":"0.0","source":"/path/to/test.js:36:6","message":"db.connect()","timestamp":"2018-07-25T23:47:57.425Z"}
{"kind":"assertion","event":"failed","id":"0.0.0","source":"/path/to/test.js:42:6","message":"Expected:\n { port: 5432 }\nActual:\n  { port: 8000 }","timestamp":"2018-07-25T23:47:58.102Z"}
{"kind":"test","event":"failed","id":"0.0","source":"/path/to/test.js:36:6","message":"db.connect()","timestamp":"2018-07-25T23:47:58.175Z"}
{"kind":"suite","event":"failed","id":"0","source":"/path/to/test.js:32:4","message":"DatabaseConnection","timestamp":"2018-07-25T23:47:58.201Z"}
```

Or

```ts
{ kind: enum, event: enum, id: string, source: SourceLocation, message: string, timestamp: DateTime }
```

## Fields

#### `kind`

- `"suite"` A group of tests or other suites
- `"test"` An individual test case containing assertions
- `"assertion"` An individual bit of logic that passes or fails.

#### `event`

- `"started"`
- `"skipped"`
- `"passed"`
- `"failed"`
- `"info"`
- `"warning"`

You don't need to send a `"started"` event in order to send a `"passed"` or `"failed"` event.
You only really need to send it when there is a gap in time between when something began
running and when it completed. For example, a simple assertion does not need to report when
it started running.

#### `id`

A string which a series of numbers separated by periods `.` (i.e. `2.3.1.12`)

```js
{ "kind": "suite", ... "id": "0", ... }
{ "kind": "test", ... "id": "0.0", ... }
{ "kind": "assertion", ... "id": "0.0.0", ... }
{ "kind": "assertion", ... "id": "0.0.1", ... }
{ "kind": "test", ... "id": "0.0", ... }
{ "kind": "test", ... "id": "0.1", ... }
{ "kind": "assertion", ... "id": "0.1.0", ... }
{ "kind": "test", ... "id": "0.1", ... }
{ "kind": "test", ... "id": "0.2", ... }
{ "kind": "assertion", ... "id": "0.2.0", ... }
{ "kind": "assertion", ... "id": "0.2.1", ... }
{ "kind": "test", ... "id": "0.2", ... }
...
```

This is a flat way to describe a tree of suites, tests, and assertions.

```yaml
- suite: "0"
  - test: "0.0"
    - assertion: "0.0.0"
    - assertion: "0.0.1"
  - test: "0.1"
    - assertion: "0.1.0"
  - test: "0.2"
    - assertion: "0.2.0"
    - assertion: "0.2.1"
- suite: "1"
  - test: "1.0"
    - assertion: "1.0.0"
- suite: "2"
  - test: "2.0"
    - assertion: "2.0.0"
    - assertion: "2.0.1"
```

#### `source`

A string of a file path and optionally line and column numbers.

```js
{ ... "source": "/path/to/project/test.js" ... } // filepath
{ ... "source": "/path/to/project/test.js:42" ... } // filepath:line
{ ... "source": "/path/to/project/test.js:42:6" ... } // filepath:line:column
```

File paths should be `/absolute`, lines are 1-index based, columns are 0-index based

#### `message`

A string describing the event.

```js
{ "kind": "suite" ... "message": "DatabaseConnection" ... }
{ "kind": "test" ... "message": "db.connect()" ... }
{ "kind": "assertion" ... "message": "Expected:\n { port: 5432 }\nActual:\n  { port: 8000 }" ... }
```

#### `timestamp`

A string which is a valid ISO 8601 date and time

```js
{ ... "timestamp": "2018-07-25T23:47:57.425Z" }
```

Timestamps are used to calculate durations of suites, tests, or assertions. For example, if
you have a test's `"started"` event and its `"passed"` event, you can compare their
timestamps in order to tell how long the test took.
