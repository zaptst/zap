# ZAP ("Ze'report Anything Protocol")

> A streamable structured interface for real time reporting of developer tools.

**Important!** This specification is a work in progress and is seeking feedback
from potential users and any developer tooling authors. It may change
dramatically based on feedback.

## Motivation

There are thousands of different developer tools across every programming
language. Each of them has their own way of reporting the results of tests
while they are running and after they are completed.

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
{"kind":"group","event":"started","id":"0","time":304.09246500208974,"content":[{"message":"DatabaseConnection","source":[{"file":"/path/to/test.js","start":{"line":3,"column":6},"end":{"line":3,"column":26}}]}]}
{"kind":"item","event":"started","id":"0.0","time":318.4295700006187,"content":[{"message":"db.connect()","source":[{"file":"/path/to/test.js","start":{"line":4,"column":8},"end":{"line":4,"column":20}}]}]}
{"kind":"check","event":"failed","id":"0.0.0","time":379.8125419989228,"content":[{"message":"Expected:\n { port: 5432 }\nActual:\n  { port: 8000 }","source":[{"file":"/path/to/test.js","start":{"line":42,"column":6}}]}]}
{"kind":"item","event":"failed","id":"0.0","time":390.98526199907064,"content":[{"message":"db.connect()","source":[{"file":"/path/to/test.js","start":{"line":4,"column":8},"end":{"line":4,"column":20}}]}]}
{"kind":"group","event":"failed","id":"0","time":464.20705600082874,"content":[{"message":"DatabaseConnection","source":[{"file":"/path/to/test.js","start":{"line":3,"column":6},"end":{"line":3,"column":26}}]}]}
```

Individual events have this shape:

```ts
interface Event {
  kind: string,
  event: string,
  id: string,
  time: number,
  status?: string,
  content: Array<{
    message: string,
    source?: Array<{
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

- `"group"` Contains `"group"`'s, `"item"`'s, or `"check"`'s, passes or fails
  based on children.
- `"item"` Contains `"check"`'s, passes or fails based on children.
- `"check"` Represents an individual assertion that either passes or fails.

You can have any of these (including `"check"`'s) with top-level `"id"`'s, you
don't have to wrap everything in a `"group"` or `"item"`.

Note that if a `"group"` or `"item"` fails, they should always have at least
one child kind that failed. i.e. You should always have at least one reported
`"check"` that failed.

#### Kind Examples

The structure of reported event kinds may differ a lot based on the tool,

##### Testing Framework

```yaml
- item: Test
  - check: Assertion
- group: Suite
  - item: Test
    - check: Assertion
    - check: Assertion
  - item: Test
    - check: Assertion
  - group: Nested Suite
    - item: Test
      - check: Assertion
      - check: Assertion
      - check: Assertion
```

#### Linter

```yaml
- group: File 1
  - check: Lint Error
  - check: Lint Error
- group: File 2
  - check: Lint Error
  - check: Lint Error
```

#### Type Checker

```yaml
- check: Type Error
- check: Type Error
- check: Type Error
- check: Type Error
- check: Type Error
- check: Type Error
```

### `event`

The name of the event.

- `"started"` Execution has started.
- `"info"` Still executing, but has some information.
- `"completed"` Execution has completed.

You don't need to send a `"started"` event in order to send a `"completed"`
event. You only really need to send it when there is a gap in time between when
something began running and when it completed. For example, a simple check does
not need to report when it started running.

### `status`

- `"running"`
- `"passed"`
- `"failed"`
- `"errored"`
- `"skipped"`

All statuses except for `"running"` should be considered final. They should
only be updated by restarting an entity.

```yaml
# Bad
- 0: started (running)
- 0: info (failed)
- 0: completed (passed)

# Good
- 0: started (running)
- 0: completed (failed)
- 0: started (running) "Retry"
- 0: completed (passed)

# Bad
- 0: started (running)
  - 0.0: started (running)
  - 0.0: completed (passed)
  - 0.1: started (running)
  - 0.1: completed (failed)
- 0: completed (failed)
  - 0.1: started (running)
  - 0.1: completed (passed) "Retry"
- 0: completed (passed)

# Good
- 0: started (running)
  - 0.0: started (running)
  - 0.0: completed (passed)
  - 0.1: started (running)
  - 0.1: completed (failed)
- 0: completed (failed)
- 0: started (running) "Retry"
  - 0.1: started (running)
  - 0.1: completed (passed) "Retry"
- 0: completed (passed)
```

A parent's status should be `"passed"` when all children `"passed"`:

```yaml
- 0: passed
  - 0.0: passed
  - 0.1: passed
```

A parent may have a status of `"passed"` or `"failed"` when a child was
`"skipped"`. Skips may have multiple different reasons. They are not all
failures depending on the situation.

```yaml
- 0: passed
  - 0.0: passed
  - 0.1: skipped
- 1: failed
  - 1.0: passed
  - 1.1: skipped
```

But when one or more children fail or error, you *must* fail the parent.

```yaml
- 0: failed
  - 0.0: failed
  - 0.1: passed
- 2: failed
  - 2.0: errored
  - 2.1: passed
- 1: failed
  - 1.0: failed
  - 1.1: errored
```

Inversely, if you may not fail a parent if all the children passed, express
your failure with another `"check"`. Although it can error.

```yaml
# Bad
- 0: failed
  - 0.0: passed
  - 0.1: passed

# Good
- 0: failed
  - 0.0: passed
  - 0.1: passed
  - 0.2: errored "An error occurred when cleaning up the database"

# Good
- 0: errored "An error occurred when cleaning up the database"
  - 0.0: passed
  - 0.1: passed
```

A parent can also choose to update their status to `"failed"` early when a
child has failed and there are more children still running.

```yaml
- 0: failed
  - 0.0: failed
  - 0.1: running
  - 0.2: running
```

### `id`

The identifier for the entity.

A string which a series of numbers separated by periods `.` (i.e. `2.3.1.12`)

```js
{ "kind": "group", ... "id": "0", ... }
{ "kind": "item", ... "id": "0.0", ... }
{ "kind": "check", ... "id": "0.0.0", ... }
{ "kind": "check", ... "id": "0.0.1", ... }
{ "kind": "item", ... "id": "0.0", ... }
{ "kind": "item", ... "id": "0.1", ... }
{ "kind": "check", ... "id": "0.1.0", ... }
{ "kind": "item", ... "id": "0.1", ... }
{ "kind": "item", ... "id": "0.2", ... }
{ "kind": "check", ... "id": "0.2.0", ... }
{ "kind": "check", ... "id": "0.2.1", ... }
{ "kind": "item", ... "id": "0.2", ... }
...
```

This is a flat way to describe a tree of groups, items, and checks.

```yaml
- group: "0"
  - item: "0.0"
    - check: "0.0.0"
    - check: "0.0.1"
  - item: "0.1"
    - check: "0.1.0"
  - item: "0.2"
    - check: "0.2.0"
    - check: "0.2.1"
- group: "1"
  - item: "1.0"
    - check: "1.0.0"
- group: "2"
  - item: "2.0"
    - check: "2.0.0"
    - check: "2.0.1"
```

You can determine the "parent" of the entity by stripping the last `.number`
(i.e. `s/\.\d+$//`)

### `time`

The offset time the event occured.

A number in milliseconds with arbitrary precision up to nanoseconds.

```js
{ ... "time": 257.0332779996097  ... }
{ ... "time": 304.6154760001227  ... }
{ ... "time": 397.152665999718   ... }
{ ... "time": 425.59379499964416 ... }
{ ... "time": 496.1821520002559  ... }
```

To calculate the duration of an entity you can compare the entity's `"start"`
and `"completed"` events.

```js
{ "id": "0.0", "event": "started", "time": 304.6154760001227 ... }
{ "id": "0.0", "event": "completed", "time": 425.59379499964416 ... }
```

```
425.5937949996ms - 304.6154760001227ms = 120.9783189995ms = 0.12s
```

**Important!** You should not assume that event times from entities with
different `"id"`'s have anything to do with one another. They could have been
run in separate parallel processes, or even across different machines.

```js
{ "id": "1.8", "event": "started", "time": 304.6154760001227 ... } // from process 1
{ "id": "4.3", "event": "started", "time": 4235.59379499964416 ... } // from process 2
{ "id": "2.2", "event": "started", "time": 204.152665999718 ... } // from process 3
// none of these times have anything to do with one another
```

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
