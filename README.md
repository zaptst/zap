# ZAP ("Ze'test Anything Protocol")

> A streamable structured interface for reporting tests in real time.

**Important!** This specification is a work in progress and is seeking feedback
from potential users such a testing framework authors and CI/CD providers. It
may change dramatically based on feedback.

## Motivation

There are hundreds of different testing frameworks across every programming
language. Each of them has their own way of reporting the results of tests
while they are runnning and after they are completed.

But we need some standard formats for reporting test results so that tools can
work together without having to support hundreds of different formats for every
language.

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

**JUnit** is easy to parse and lots of structured information about tests, but
cannot be streamed as tests run. You have to wait until your tests are finished
before generating the XML.

**TAP** is great for streaming test results as they run and can be used to
report progress to developers. But it is hard to parse and contains very little
structured information.

**ZAP** is a new reporting format that tries to get the best of both systems, it
contains well structured information about tests and can be streamed and parsed
very easily.

#### Goals

- **Streaming:** Allow test frameworks to report test information to consumers
  in real time as tests execute.
- **Concurrency:** Allow tests to be run concurrently across threads or
  processes and allow interweaving of events in output.
- **Structured:** Use a well-defined structure that is easy to parse and
  understand with lots of information.
- **Extensible:** Have a clear way of adding additional information to test
  output that is still meets the above goals.

#### Non-goals

- **Readability:** Where TAP is readable, ZAP does not have to be. The idea is
  that like TAP it'd be easy to build reporters on top of ZAP.

#### Comparison

|                 | XML | TAP | ZAP |
| ---------------:|:---:|:---:|:---:|
| **Streaming**   |  ❌  |  ✅  |  ✅  |
| **Concurrency** | N/A |  👌  |  ✅  |
| **Structured**  |  ✅  |  ❌  |  ✅  |
| **Extensible**  |  ✅  |  ❌  |  ✅  |
| **Readable**    |  ❌  |  ✅  |  ❌  |

## Spec

### Format

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

### Fields

#### `kind`

The type of entity the event is for.

- `"suite"` A group of tests or other suites
- `"test"` An individual test case containing assertions
- `"assertion"` An individual bit of logic that passes or fails.

#### `event`

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

#### `id`

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

#### `source`

The source location for the event.

A string of a file path and optionally line and column numbers.

```js
{ ... "source": "/path/to/project/test.js" ... } // filepath
{ ... "source": "/path/to/project/test.js:42" ... } // filepath:line
{ ... "source": "/path/to/project/test.js:42:6" ... } // filepath:line:column
```

File paths should be absolute, lines are 1-index based, columns are 0-index
based.

#### `message`

The description of the event.

```js
{ "kind": "suite" ... "message": "DatabaseConnection" ... }
{ "kind": "test" ... "message": "db.connect()" ... }
{ "kind": "assertion" ... "message": "Expected:\n { port: 5432 }\nActual:\n  { port: 8000 }" ... }
```

#### `timestamp`

The time the event occurred.

A valid ISO 8601 date and time string.

```js
{ ... "timestamp": "2018-07-25T23:47:57.425Z" }
```

Timestamps are used to calculate durations of suites, tests, or assertions. For
example, if you have a test's `"started"` event and its `"passed"` event, you
can compare their timestamps in order to tell how long the test took.
