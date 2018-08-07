# ZAP ("Ze'test Anything Protocol")

> A streamable structured interface for real time reporting of developer tools.

**Important!** This specification is a work in progress and is seeking feedback
from potential users and any developer tooling authors. It may change
dramatically based on feedback.

## Motivation

There are thousands of different developer tools across every programming
language. Each of them has their own way of reporting the results of tests
while they are runnning and after they are completed.

But we need some standard formats for reporting their results so that tools can
work together without having to support hundreds of different formats for every
language and tool.

Today there exists two prominent formats for reporting test results:

**JUnit XML**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<testsuites id="20140612_170519" name="New_configuration (14/06/12 17:05:19)" tests="225" failures="1262" time="0.001">
  <testsuite id="codereview.cobol.analysisProvider" name="COBOL Code Review" tests="45" failures="17" time="0.001">
    <testcase id="codereview.cobol.rules.ProgramIdRule" name="Use a program name that matches the source file name" time="0.001">
      <failure message="PROGRAM.cbl:2 Use a program name that matches the source file name" type="WARNING">
WARNING: Use a program name that matches the source file name
Category: COBOL Code Review – Naming Conventions
File: /project/PROGRAM.cbl
Line: 2
      </failure>
    </testcase>
  </testsuite>
</testsuites>
```

**Test Anything Protocol (aka "TAP")**

```tap
1..4
ok 1 - Input file opened
not ok 2 - First line of the input valid
ok 3 - Read the rest of the file
not ok 4 - Summarized correctly # TODO Not written yet
```

Each of these has different strengths and weaknesses.

**JUnit** is easy to parse and lots of structured information about results, but
cannot be streamed as tests run. You have to wait until your tests are finished
before generating the XML.

**TAP** is great for streaming test results as they run and can be used to
report progress to developers. But it is hard to parse and contains very little
structured information.

**ZAP** is a new reporting format that tries to get the best of both systems, it
contains well structured information about results and can be streamed and
parsed very easily.

#### Goals

- **Streaming:** Allow developer tools to report test information to consumers
  in real time as tests execute.
- **Concurrency:** Allow tools to be run concurrently across threads or
  processes and allow interweaving of events in output.
- **Structured:** Use a well-defined structure that is easy to parse and
  understand with lots of information.
- **Definitive:** At any given point, it should be clear what state things are
  in: "Passing" or "Failing". When a completion state is given, it should not
  change.
- **Extensible:** Have a clear way of adding additional information to output
  that is still meets the above goals.

#### Non-goals

- **Readability:** It should be easy to build reporting tools on top of ZAP
  that is human readable, but ZAP itself should be designed for machines.

#### Comparison

|                 | XML | TAP | ZAP |
| ---------------:|:---:|:---:|:---:|
| **Streaming**   |  ❌  |  ✅  |  ✅  |
| **Concurrency** | N/A |  👌  |  ✅  |
| **Structured**  |  ✅  |  ❌  |  ✅  |
| **Extensible**  |  ✅  |  ❌  |  ✅  |
| **Readable**    |  ❌  |  ✅  |  ❌  |

#### Considerations

There are many different types of development tools that would like a standard
way to report results via a format like ZAP.

They generally fall into one of two categories:

1. **Producers**
  - Testing Frameworks
  - Linters/Code Quality
  - Type Checkers
  - Compilers/Build Tools
2. **Consumers**
  - CI/CD Platforms/Services
  - Task Runners/Build Frameworks
  - Reporters (CLI or GUI)

Many of these tools already use JUnit or TAP, however many are unhappy with
these formats and would like something better. ZAP should consider all of these
tools and provide guidance on implementations.

## Spec

### Format

The output is stream of events written in newline-delimited json:

```js
{"kind":"suite","event":"started","id":"0","timestamp":"2018-07-25T23:47:57.133Z","content":[{"message":"DatabaseConnection","source":[{"file":"/path/to/test.js","start":{"line":3,"column":6},"end":{"line":3,"column":26}}]}]}
{"kind":"test","event":"started","id":"0.0","timestamp":"2018-07-25T23:47:57.425Z","content":[{"message":"db.connect()","source":[{"file":"/path/to/test.js","start":{"line":4,"column":8},"end":{"line":4,"column":20}}]}]}
{"kind":"assertion","event":"failed","id":"0.0.0","timestamp":"2018-07-25T23:47:58.102Z","content":[{"message":"Expected:\n { port: 5432 }\nActual:\n  { port: 8000 }","source":[{"file":"/path/to/test.js","start":{"line":42,"column":6}}]}]}
{"kind":"test","event":"failed","id":"0.0","timestamp":"2018-07-25T23:47:58.175Z","content":[{"message":"db.connect()","source":[{"file":"/path/to/test.js","start":{"line":4,"column":8},"end":{"line":4,"column":20}}]}]}
{"kind":"suite","event":"failed","id":"0","timestamp":"2018-07-25T23:47:58.201Z","content":[{"message":"DatabaseConnection","source":[{"file":"/path/to/test.js","start":{"line":3,"column":6},"end":{"line":3,"column":26}}]}]}
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
      start?: { line: number, column?: number },
      end?: { line: number, column?: number }
    }>
  }>
}
```

### Fields

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
    { "message": "number", "source": [{ "file": "/path/to/source/file.js", "start": { "line": 15, "column": 12 }, "end": { "line": 15, "column": 19 } }] },
    { "message": "is not compatible with" },
    { "message": "string", "source": [{ "file": "/path/to/source/file.js", "start": { "line": 19, "column": 8 }, "end": { "line": 19, "column": 14 } }] }
  ]
  ...
}
```

Every element in the `"content"` array should be an object with the following
shape:

```ts
interface ContentPart {
  message: string,
  source?: Array<{
    file: string,
    start?: { line: number, column?: number },
    end?: { line: number, column?: number }
  }>
}
```

#### Content Targets

When interpreting elements of `"content"` you should not attempt to "fill in"
the missing elements.

- If there is no `"source"`, do not make one up or try associating
  it with the other elements in the `"content"` array.
- If there is a `"file"` but no `"start"` or `"end"` positions, assume it means
  the file itself, not the range of the file's content.
- If there is a `"line"` but no `"column"`, assume it means the entire line,
  not range of the line's content.
- If there is a `"start"` but no `"end"`, assume it means that exact position,
  not a range of a single characters, or to the rest of the file.

#### Content Positions

When interpreting a position:

- `"line"` is 1-index based
- `"column"` is 0-index based


This means that when you are interpreting a `"column"` it is the index *before*
a character. It targets this "in between" position and not the character itself.

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
    "source": [{
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
    "source": [{
      "file": "/path/to/file.js",
      "start": { "line": 10000 }
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
    "source": [{
      "file": "/path/to/file.js",
      "start": { "line": 24, "column": 4 }
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
    "source": [{
      "file": "/path/to/file.js",
      "start": { "line": 12 },
      "end": { "line": 14 }
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
    "source": [{
      "file": "/path/to/file.js",
      "start": { "line": 9, "column": 6 },
      "end": { "line": 9, "column": 13 }
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
    "source": [{
      "file": "/path/to/file.js",
      "start": { "line": 9, "column": 6 },
      "end": { "line": 9, "column": 13 }
    }, {
      "file": "/path/to/file.js",
      "start": { "line": 27, "column": 6 },
      "end": { "line": 27, "column": 13 }
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
      "source": [{
        "file": "/path/to/one.js",
        "start": { "line": 15, "column": 38 },
        "end": { "line": 15, "column": 45 }
      }]
    },
    {
      "message": "is not compatible with"
    },
    {
      "message": "string",
      "source": [{
        "file": "/path/to/two.js",
        "start": { "line": 9, "column": 6 },
        "end": { "line": 9, "column": 13 }
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
