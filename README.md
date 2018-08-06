# ZAP ("Ze'test Anything Protocol")

> A streamable structured interface for reporting tests in real time.

## Format

The output is stream of events written in newline-delimited json:

```js
{"kind":"suite","event":"started","id":"0","timestamp":"2018-07-25T23:47:57.133Z","content":[{"message":"DatabaseConnection","loc":[{"file":"/path/to/test.js","start":{"row":3,"col":6},"end":{"row":3,"col":26}}]}]}
{"kind":"test","event":"started","id":"0.0","timestamp":"2018-07-25T23:47:57.425Z","content":[{"message":"db.connect()","loc":[{"file":"/path/to/test.js","start":{"row":4,"col":8},"end":{"row":4,"col":20}}]}]}
{"kind":"assertion","event":"failed","id":"0.0.0","timestamp":"2018-07-25T23:47:58.102Z","content":[{"message":"Expected:\n { port: 5432 }\nActual:\n  { port: 8000 }","loc":[{"file":"/path/to/test.js","start":{"row":42,"col":6}}]}]}
{"kind":"test","event":"failed","id":"0.0","timestamp":"2018-07-25T23:47:58.175Z","content":[{"message":"db.connect()","loc":[{"file":"/path/to/test.js","start":{"row":4,"col":8},"end":{"row":4,"col":20}}]}]}
{"kind":"suite","event":"failed","id":"0","timestamp":"2018-07-25T23:47:58.201Z","content":[{"message":"DatabaseConnection","loc":[{"file":"/path/to/test.js","start":{"row":3,"col":6},"end":{"row":3,"col":26}}]}]}
```

Individual events have this shape:

```ts
interface Event {
  kind: string,
  event: string,
  id: string,
  timestamp: string,
  content: Array<{
    message: string,
    loc?: Array<{
      file: string,
      start?: { row: number, col?: number },
      end?: { row: number, col?: number }
    }>
  }>
}
```

## Fields

### `kind`

The type of entity the event is for.

- `"suite"` A group of tests or other suites
- `"test"` An individual test case containing assertions
- `"assertion"` An individual bit of logic that passes or fails.

### `event`

The name of the event.

- `"started"` Execution has started.
- `"passed"` Execution has completed, and it was all successful.
- `"failed"` Execution has completed, and one or more parts of it failed.
- `"skipped"` Execution was skipped.
- `"info"` A info message.
- `"warning"` A warning message.

You don't need to send a `"started"` event in order to send a `"passed"` or
`"failed"` event. You only really need to send it when there is a gap in time
between when something began running and when it completed. For example, a
simple assertion does not need to report when it started running.

### `id`

The identifier for the entity.

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

You can determine the "parent" of the entity by stripping the last `.number`
(i.e. `s/\.\d+$//`)

### `timestamp`

The time the event occurred.

A valid ISO 8601 date and time string.

```js
{ ... "timestamp": "2018-07-25T23:47:57.425Z" }
```

Timestamps are used to calculate durations of suites, tests, or assertions. For
example, if you have a test's `"started"` event and its `"passed"` event, you
can compare their timestamps in order to tell how long the test took.

### `content`

The content of the event.

An array of messages and (optionally) source locations.

```js
{ ...
  "content": [
    { "message": "number", "loc": [{ "file": "/path/to/source/file.js", "start": { "row": 15, "col": 12 }, "end": { "row": 15, "col": 19 } }] },
    { "message": "is not compatible with" },
    { "message": "string", "loc": [{ "file": "/path/to/source/file.js", "start": { "row": 19, "col": 8 }, "end": { "row": 19, "col": 14 } }] }
  ]
  ...
}
```

Every element in the `"content"` array should be an object with the following shape

```ts
interface ContentPart {
  message: string,
  loc?: Array<{
    file: string,
    start?: { row: number, col?: number },
    end?: { row: number, col?: number }
  }>
}
```

#### Content Targets

When interpreting elements of `"content"` you should not attempt to "fill in"
the missing elements.

- If there is no `"loc"`, do not make one up or try associating
  it with the other elements in the `"content"` array.
- If there is a `"file"` but no `"start"` or `"end"` positions, assume it means
  the file itself, not the range of the file's content.
- If there is a `"row"` but no `"col"`, assume it means the entire row, not
  range of the row's content.
- If there is a `"start"` but no `"end"`, assume it means that exact position,
  not a range of a single characters, or to the rest of the file.

#### Content Positions

When interpreting a position:

- `"row"` is 1-index based
- `"col"` is 0-index based


This means that when you are interpreting a `"col"` it is the index *before* a
character. It targets this "in between" position and not the character itself.

For example if we have the following line:

```
012345
```

And we are selecting columns 1 through 4, we would get the following selection:

```
0[123]45
  ^^^
```

#### Content Examples

##### A message with no location:

```js
{ ...
  "content": [{
    "message": "No tests found"
  }]
}
```

```
No tests found
```

##### An entire file:

```js
{ ...
  "content": [{
    "message": "File named incorectly",
    "loc": [{
      "file": "/path/to/file.js"
    }]
  }]
}
```

```
/path/to/file.js: File named incorrectly
```

##### A specific line in a file:

```js
{ ...
  "content": [{
    "message": "File too long",
    "loc": [{
      "file": "/path/to/file.js",
      "start": { "row": 10000 }
    }]
  }]
}
```

```
/path/to/file.js
   9998 |   "lorem",
   9999 |   "ipsum",
> 10000 |   "dolor",
  10001 |   "sit"
  10002 |   "amet",
File too long
```

##### A specific position in a file:

```js
{ ...
  "content": [{
    "message": "Missing closing brace",
    "loc": [{
      "file": "/path/to/file.js",
      "start": { "row": 24, "col": 4 }
    }]
  }]
}
```

```
/path/to/file.js
  22 |     for (let item of items) {
  23 |       total += item.price;
> 24 |
     |     ^ Missing closing brace
  25 |     return toPriceString(total);
  26 |   }
```

##### A range of lines in a file:

```js
{ ...
  "content": [{
    "message": "...",
    "loc": [{
      "file": "/path/to/file.js",
      "start": { "row": 12 },
      "end": { "row": 14 }
    }]
  }]
}
```

```
/path/to/file.js
  11 |   ...
> 12 |     ...
> 13 |     ...
> 14 |     ...
  15 |   ...
...
```

##### A range in a file:

```js
{ ...
  "content": [{
    "message": "Variable name `public` is reserved word",
    "loc": [{
      "file": "/path/to/file.js",
      "start": { "row": 9, "col": 6 },
      "end": { "row": 9, "col": 13 }
    }]
  }]
}
```

```
/path/to/file.js
   7 |
   8 |   function isPublic(item) {
>  9 |     let public = item.public;
     |         ^^^^^^ Variable name `public` is reserved word
  10 |     let childrenPublic = item.children.every(child => {
  11 |       return isPublic(child);
```

##### Multiple ranges in a file:

```js
{ ...
  "content": [{
    "message": "Binding `active` declared multiple times",
    "loc": [{
      "file": "/path/to/file.js",
      "start": { "row": 9, "col": 6 },
      "end": { "row": 9, "col": 13 }
    }, {
      "file": "/path/to/file.js",
      "start": { "row": 27, "col": 6 },
      "end": { "row": 27, "col": 13 }
    }]
  }]
}
```

```
/path/to/file.js
   8 |   function isActive(item) {
>  9 |     let active = item.active;
     |         ^^^^^^
  10 |     let childrenActive = item.children.every(child => {
    ...
  26 |
  27 |     let active = item.children.filter(child => {
     |         ^^^^^^
  28 |       return isActive(child);
Binding `active` declared multiple times
```

##### Multi-part message

```js
{ ...
  "content": [
    {
      "message": "number",
      "loc": [{
        "file": "/path/to/one.js",
        "start": { "row": 15, "col": 38 },
        "end": { "row": 15, "col": 45 }
      }]
    },
    {
      "message": "is not compatible with"
    },
    {
      "message": "string",
      "loc": [{
        "file": "/path/to/two.js",
        "start": { "row": 9, "col": 6 },
        "end": { "row": 9, "col": 13 }
      }]
    }
  }]
}
```

```
/path/to/one.js
  14 |
> 15 | export function calculateTotal(item): number {
     |                                       ^^^^^^ number
  16 |

is not compatible with

/path/to/two.js
  36 |     let message = 'Total: ';
> 37 |     let total: string = calculateTotal(item);
     |                ^^^^^^ string
  38 |     message += total;
```
