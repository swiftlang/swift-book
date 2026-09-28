# Concurrency

Perform asynchronous operations.

Swift has built-in support for writing asynchronous and parallel code
in a structured way.
*Asynchronous code* can be suspended and resumed later,
although only one piece of the program executes at a time.
Suspending and resuming code in your program
lets it continue to make progress
on short-term operations like updating its UI
while continuing to work on long-running operations
like fetching data over the network or parsing files.
*Parallel code* means multiple pieces of code run simultaneously ---
for example, a computer with a four-core processor
can run four pieces of code at the same time,
with each core carrying out one of the tasks.
A program that uses parallel and asynchronous code
carries out multiple operations at a time,
and it suspends operations that are waiting for an external system.
The rest of this chapter uses the term *concurrency*
to refer to this common combination of asynchronous and parallel code.

The additional scheduling flexibility from parallel or asynchronous code
also comes with a cost of increased complexity.
When you write concurrent code,
you don't know ahead of time what code will run at the same time,
and you might not always know the order that code will run.
A common problem in concurrent code happens
when multiple pieces of code try to access
some piece of shared mutable state ---
this is known as a *data race*.
When you use the language-level support for concurrency,
Swift detects and prevents data races,
and most data races produce a compile-time error.
Some data races can't be detected until your code is running;
these data races terminate code execution.
You use actors and isolation to protect against data races,
as described in this chapter.

> Note: If you've written concurrent code before,
> you might be used to working with threads.
> The concurrency model in Swift is built on top of threads,
> but you don't interact with them directly.
> An asynchronous function in Swift
> can give up the thread that it's running on,
> which lets another asynchronous function run on that thread
> while the first function is blocked.
> When an asynchronous function resumes,
> Swift doesn't make any guarantee about which thread
> that function will run on.

Although it's possible to write concurrent code
without using Swift's language support,
that code tends to be harder to read.
For example, the following code downloads a list of photo names,
downloads the first photo in that list,
and shows that photo to the user:

```swift
listPhotos(inGallery: "Summer Vacation") { photoNames in
    let sortedNames = photoNames.sorted()
    let name = sortedNames[0]
    downloadPhoto(named: name) { photo in
        show(photo)
    }
}
```

<!--
  - test: `async-via-nested-completion-handlers`

  ```swifttest
  >> struct Data {}  // Instead of actually importing Foundation
  >> func listPhotos(inGallery name: String, completionHandler: ([String]) -> Void ) {
  >>   completionHandler(["IMG001", "IMG99", "IMG0404"])
  >> }
  >> func downloadPhoto(named name: String, completionHandler: (Data) -> Void) {
  >>     completionHandler(Data())
  >> }
  >> func show(_ image: Data) { }
  -> listPhotos(inGallery: "Summer Vacation") { photoNames in
         let sortedNames = photoNames.sorted()
         let name = sortedNames[0]
         downloadPhoto(named: name) { photo in
             show(photo)
         }
     }
  ```
-->

Even in this simple case,
because the code has to be written as a series of completion handlers,
you end up writing nested closures.
In this style,
more complex code with deep nesting can quickly become unwieldy.

## Defining and Calling Asynchronous Functions

An *asynchronous function* or *asynchronous method*
is a special kind of function or method
that can be suspended while it's partway through execution.
This is in contrast to ordinary, synchronous functions and methods,
which either run to completion, throw an error, or never return.
An asynchronous function or method still does one of those three things,
but it can also pause in the middle when it's waiting for something.
Inside the body of an asynchronous function or method,
you mark each of these places where execution can be suspended.

To indicate that a function or method is asynchronous,
you write the `async` keyword in its declaration after its parameters,
similar to how you use `throws` to mark a throwing function.
If the function or method returns a value,
you write `async` before the return arrow (`->`).
For example,
here's how you might fetch the names of photos in a gallery:

```swift
func listPhotos(inGallery name: String) async -> [String] {
    let result = // ... some asynchronous networking code ...
    return result
}
```

<!--
  - test: `async-function-shape`

  ```swifttest
  -> func listPhotos(inGallery name: String) async -> [String] {
         let result = // ... some asynchronous networking code ...
  >>     ["IMG001", "IMG99", "IMG0404"]
         return result
     }
  ```
-->

For a function or method that's both asynchronous and throwing,
you write `async` before `throws`.

<!--
  - test: `async-comes-before-throws`

  ```swifttest
  >> func right() async throws -> Int { return 12 }
  >> func wrong() throws async -> Int { return 12 }
  !$ error: 'async' must precede 'throws'
  !! func wrong() throws async -> Int { return 12 }
  !! ^~~~~~
  !! async
  ```
-->

When calling an asynchronous method,
execution suspends until that method returns.
You write `await` in front of the call
to mark the possible suspension point.
This is like writing `try` when calling a throwing function,
to mark the possible change to the program's flow if there's an error.
Inside an asynchronous method,
the flow of execution can be suspended
*only* when you call another asynchronous method ---
suspension is never implicit or preemptive ---
which means every possible suspension point is marked with `await`.
Marking all of the possible suspension points in your code
helps make concurrent code easier to read and understand.

For example,
the code below fetches the names of all the pictures in a gallery
and then shows the first picture:

```swift
let photoNames = await listPhotos(inGallery: "Summer Vacation")
let sortedNames = photoNames.sorted()
let name = sortedNames[0]
let photo = await downloadPhoto(named: name)
show(photo)
```

<!--
  - test: `defining-async-function`

  ```swifttest
  >> struct Data {}  // Instead of actually importing Foundation
  >> func downloadPhoto(named name: String) async -> Data { return Data() }
  >> func show(_ image: Data) { }
  >> func listPhotos(inGallery name: String) async -> [String] {
  >>     return ["IMG001", "IMG99", "IMG0404"]
  >> }
  >> func f() async {
  -> let photoNames = await listPhotos(inGallery: "Summer Vacation")
  -> let sortedNames = photoNames.sorted()
  -> let name = sortedNames[0]
  -> let photo = await downloadPhoto(named: name)
  -> show(photo)
  >> }
  ```
-->

Because the `listPhotos(inGallery:)` and `downloadPhoto(named:)` functions
both need to make network requests,
they could take a relatively long time to complete.
Making them both asynchronous by writing `async` before the return arrow
lets the rest of the app's code keep running
while this code waits for the picture to be ready.

To understand the concurrent nature of the example above,
here's one possible order of execution:

1. The code starts running from the first line
   and runs up to the first `await`.
   It calls the `listPhotos(inGallery:)` function
   and suspends execution while it waits for that function to return.

2. While this code's execution is suspended,
   some other concurrent code in the same program runs.
   For example, maybe a long-running background task
   continues updating a list of new photo galleries.
   That code also runs until the next suspension point, marked by `await`,
   or until it completes.

3. After `listPhotos(inGallery:)` returns,
   this code continues execution starting at that point.
   It assigns the value that was returned to `photoNames`.

4. The lines that define `sortedNames` and `name`
   are regular, synchronous code.
   Because nothing is marked `await` on these lines,
   there aren't any possible suspension points.

5. The next `await` marks the call to the `downloadPhoto(named:)` function.
   This code pauses execution again until that function returns,
   giving other concurrent code an opportunity to run.

6. After `downloadPhoto(named:)` returns,
   its return value is assigned to `photo`
   and then passed as an argument when calling `show(_:)`.

The possible suspension points in your code marked with `await`
indicate that the current piece of code might pause execution
while waiting for the asynchronous function or method to return.
This is also called *yielding the thread*
because, behind the scenes,
Swift suspends the execution of your code on the current thread
and runs some other code on that thread instead.
Because code with `await` needs to be able to suspend execution,
only certain places in your program can call asynchronous functions or methods:

- Code in the body of an asynchronous function, method, or property.

- Code in the static `main()` method of
  a structure, class, or enumeration that's marked with `@main`.

- Code in an unstructured child task,
  as shown in <doc:Concurrency#Unstructured-Concurrency> below.

<!--
  SE-0296 specifically calls out that top-level code is *not* an async context,
  contrary to what you might expect.
  If that gets changed, add this bullet to the list above:

  - Code at the top level that forms an implicit main function.
-->

The [`Task.sleep(for:tolerance:clock:)`][] method
is useful when writing simple code
to learn how concurrency works.
This method suspends the current task for at least the given amount of time.
Here's a version of the `listPhotos(inGallery:)` function
that uses `sleep(for:tolerance:clock:)` to simulate waiting for a network operation:

[`Task.sleep(for:tolerance:clock:)`]: https://developer.apple.com/documentation/swift/task/sleep(for:tolerance:clock:)

```swift
func listPhotos(inGallery name: String) async throws -> [String] {
    try await Task.sleep(for: .seconds(2))
    return ["IMG001", "IMG99", "IMG0404"]
}
```

<!--
  - test: `sleep-in-toy-code`

  ```swifttest
  >> struct Data {}  // Instead of actually importing Foundation
  -> func listPhotos(inGallery name: String) async throws -> [String] {
         try await Task.sleep(for: .seconds(2))
         return ["IMG001", "IMG99", "IMG0404"]
  }
  ```
-->

The version of `listPhotos(inGallery:)` in the code above
is both asynchronous and throwing,
because the call to `Task.sleep(until:tolerance:clock:)` can throw an error.
When you call this version of `listPhotos(inGallery:)`,
you write both `try` and `await`:

```swift
let photos = try await listPhotos(inGallery: "A Rainy Weekend")
```

Asynchronous functions have some similarities to throwing functions:
When you define an asynchronous or throwing function,
you mark it with `async` or `throws`,
and you mark calls to that function with `await` or `try`.
An asynchronous function can call another asynchronous function,
just like a throwing function can call another throwing function.

However, there's a very important difference.
You can wrap throwing code in a `do`-`catch` block to handle errors,
or use `Result` to store the error for code elsewhere to handle it.
These approaches let you call throwing functions
from nonthrowing code.
For example:

```swift
func availableRainyWeekendPhotos() -> Result<[String], Error> {
    return Result {
        try listDownloadedPhotos(inGallery: "A Rainy Weekend")
    }
}
```

In contrast,
there's no safe way to wrap asynchronous code
so you can call it from synchronous code and wait for the result.
The Swift standard library intentionally omits this unsafe functionality ---
trying to implement it yourself can lead to
problems like subtle races, threading issues, and deadlocks.
When adding concurrent code to an existing project,
work from the top down.
Specifically,
start by converting the top-most layer of code to use concurrency,
and then start converting the functions and methods that it calls,
working through the project's architecture one layer at a time.
There's no way to take a bottom-up approach,
because synchronous code can't ever call asynchronous code.

<!--
  OUTLINE

  ## Asynchronous Closures

  like how you can have an async function, a closure con be async
  if a closure contains 'await' that implicitly makes it async
  you can mark it explicitly with "async -> in"

  (discussion of @MainActor closures can probably go here too)
-->

## Asynchronous Sequences

The `listPhotos(inGallery:)` function in the previous section
asynchronously returns the whole array at once,
after all of the array's elements are ready.
Another approach
is to wait for one element of the collection at a time
using an *asynchronous sequence*.
Here's what iterating over an asynchronous sequence looks like:

```swift
import Foundation

let handle = FileHandle.standardInput
for try await line in handle.bytes.lines {
    print(line)
}
```

<!--
  - test: `async-sequence`

  ```swifttest
  -> import Foundation

  >> func f() async throws {
  -> let handle = FileHandle.standardInput
  -> for try await line in handle.bytes.lines {
         print(line)
     }
  >> }
  ```
-->

Instead of using an ordinary `for`-`in` loop,
the example above writes `for` with `await` after it.
Like when you call an asynchronous function or method,
writing `await` indicates a possible suspension point.
A `for`-`await`-`in` loop potentially suspends execution
at the beginning of each iteration,
when it's waiting for the next element to be available.

In the same way that you can use your own types in a `for`-`in` loop
by adding conformance to the [`Sequence`][] protocol,
you can use your own types in a `for`-`await`-`in` loop
by adding conformance to the [`AsyncSequence`] protocol.

[`Sequence`]: https://developer.apple.com/documentation/swift/sequence
[`AsyncSequence`]: https://developer.apple.com/documentation/swift/asyncsequence

<!--
  TODO what happened to ``Series`` which was supposed to be a currency type?
  Is that coming from Combine instead of the stdlib maybe?

  Also... need a real API that produces a async sequence.
  I'd prefer not to go through the whole process of making one here,
  since the protocol reference has enough detail to show you how to do that.
  There's nothing in the stdlib except for the AsyncFooSequence types.
  Maybe one of the other conforming types from an Apple framework --
  how about FileHandle.AsyncBytes (myFilehandle.bytes.lines) from Foundation?

  https://developer.apple.com/documentation/swift/asyncsequence
  https://developer.apple.com/documentation/foundation/filehandle

  if we get a stdlib-provided async sequence type at some point,
  rewrite the above to fit the same narrative flow
  using something like the following

  let names = await listPhotos(inGallery: "Winter Vacation")
  for await photo in Photos(names: names) {
      show(photo)
  }
-->

## Calling Asynchronous Functions in Parallel

Calling an asynchronous function with `await`
runs only one piece of code at a time.
While the asynchronous code is running,
the caller waits for that code to finish
before moving on to run the next line of code.
For example,
to fetch the first three photos from a gallery,
you could await three calls to the `downloadPhoto(named:)` function
as follows:

```swift
let firstPhoto = await downloadPhoto(named: photoNames[0])
let secondPhoto = await downloadPhoto(named: photoNames[1])
let thirdPhoto = await downloadPhoto(named: photoNames[2])

let photos = [firstPhoto, secondPhoto, thirdPhoto]
show(photos)
```

<!--
  - test: `defining-async-function`

  ```swifttest
  >> func show(_ images: [Data]) { }
  >> func ff() async {
  >> let photoNames = ["IMG001", "IMG99", "IMG0404"]
  -> let firstPhoto = await downloadPhoto(named: photoNames[0])
  -> let secondPhoto = await downloadPhoto(named: photoNames[1])
  -> let thirdPhoto = await downloadPhoto(named: photoNames[2])

  -> let photos = [firstPhoto, secondPhoto, thirdPhoto]
  -> show(photos)
  >> }
  ```
-->

This approach has an important drawback:
Although the download is asynchronous
and lets other work happen while it progresses,
only one call to `downloadPhoto(named:)` runs at a time.
Each photo downloads completely before the next one starts downloading.
However, there's no need for these operations to wait ---
each photo can download independently, or even at the same time.

To call an asynchronous function
and let it run in parallel with code around it,
write `async` in front of `let` when you define a constant,
and then write `await` each time you use the constant.

```swift
async let firstPhoto = downloadPhoto(named: photoNames[0])
async let secondPhoto = downloadPhoto(named: photoNames[1])
async let thirdPhoto = downloadPhoto(named: photoNames[2])

let photos = await [firstPhoto, secondPhoto, thirdPhoto]
show(photos)
```

<!--
  - test: `calling-with-async-let`

  ```swifttest
  >> struct Data {}  // Instead of actually importing Foundation
  >> func show(_ images: [Data]) { }
  >> func downloadPhoto(named name: String) async -> Data { return Data() }
  >> let photoNames = ["IMG001", "IMG99", "IMG0404"]
  >> func f() async {
  -> async let firstPhoto = downloadPhoto(named: photoNames[0])
  -> async let secondPhoto = downloadPhoto(named: photoNames[1])
  -> async let thirdPhoto = downloadPhoto(named: photoNames[2])

  -> let photos = await [firstPhoto, secondPhoto, thirdPhoto]
  -> show(photos)
  >> }
  ```
-->

In this example,
all three calls to `downloadPhoto(named:)` start
without waiting for the previous one to complete.
If there are enough system resources available, they can run at the same time.
None of these function calls are marked with `await`
because the code doesn't suspend to wait for the function's result.
Instead, execution continues
until the line where `photos` is defined ---
at that point, the program needs the results from these asynchronous calls,
so you write `await` to pause execution
until all three photos finish downloading.

Here's how you can think about the differences between these two approaches:

- Call asynchronous functions with `await`
  when the code on the following lines depends on that function's result.
  This creates work that is carried out sequentially.
- Call asynchronous functions with `async`-`let`
  when you don't need the result until later in your code.
  This creates work that can be carried out in parallel.
- Both `await` and `async`-`let`
  allow other code to run while they're suspended.
- In both cases, you mark the possible suspension point with `await`
  to indicate that execution will pause, if needed,
  until an asynchronous function has returned.

You can also mix both of these approaches in the same code.

## Tasks and Task Groups

A *task* is a unit of work
that can be run asynchronously as part of your program.
All asynchronous code runs as part of some task.
A task itself does only one thing at a time,
but when you create multiple tasks,
Swift can schedule them to run simultaneously.

The `async`-`let` syntax described in the previous section
implicitly creates a child task ---
this syntax works well when you already know
what tasks your program needs to run.
You can also create a task group
(an instance of [`TaskGroup`][])
and explicitly add child tasks to that group,
which gives you more control over priority and cancellation,
and lets you create a dynamic number of tasks.

[`TaskGroup`]: https://developer.apple.com/documentation/swift/taskgroup

Tasks are arranged in a hierarchy.
Each task in a given task group has the same parent task,
and each task can have child tasks.
Because of the explicit relationship between tasks and task groups,
this approach is called *structured concurrency*.
The explicit parent-child relationship between tasks has several advantages:

- In a parent task,
  you can't forget to wait for its child tasks to complete.

- When setting a higher priority on a child task,
  the parent task's priority is automatically escalated.

- When a parent task is canceled,
  each of its child tasks is also automatically canceled.

- Task-local values propagate to child tasks efficiently and automatically.

Here's another version of the code to download photos
that handles any number of photos:

```swift
await withTaskGroup(of: Data.self) { group in
    let photoNames = await listPhotos(inGallery: "Summer Vacation")
    for name in photoNames {
        group.addTask {
            return await downloadPhoto(named: name)
        }
    }

    for await photo in group {
        show(photo)
    }
}
```

The code above creates a new task group,
and then creates child tasks
to download each photo in the gallery.
Swift runs as many of these tasks concurrently as conditions allow.
As soon as a child task finishes downloading a photo,
that photo is displayed.
There's no guarantee about the order that child tasks complete,
so the photos from this gallery can be shown in any order.

> Note:
> If the code to download a photo could throw an error,
> you would call `withThrowingTaskGroup(of:returning:body:)` instead.

In the code listing above,
each photo is downloaded and then displayed,
so the task group doesn't return any results.
For a task group that returns a result,
you add code that accumulates its result
inside the closure you pass to `withTaskGroup(of:returning:body:)`.

```swift
let photos = await withTaskGroup(of: Data.self) { group in
    let photoNames = await listPhotos(inGallery: "Summer Vacation")
    for name in photoNames {
        group.addTask {
            return await downloadPhoto(named: name)
        }
    }

    var results: [Data] = []
    for await photo in group {
        results.append(photo)
    }

    return results
}
```

Like the previous example,
this example creates a child task for each photo to download it.
Unlike the previous example,
the `for`-`await`-`in` loop waits for the next child task to finish,
appends the result of that task to the array of results,
and then continues waiting until all child tasks have finished.
Finally,
the task group returns the array of downloaded photos
as its overall result.

<!--
TODO:
In the future,
we could extend the example above
to show how you can limit the number of concurrent tasks
that get spun up by a task group.
There isn't a specific guideline we can give
in terms of how many concurrent tasks to run --
it's more "profile your code, and then adjust".

See also:
https://developer.apple.com/videos/play/wwdc2023/10170?time=688

We could also show withDiscardingTaskGroup(...)
since that's optimized for child tasks
whose values aren't collected.
-->

### Task Cancellation

Swift concurrency uses a cooperative cancellation model.
Each task checks whether it has been canceled
at the appropriate points in its execution,
and responds to cancellation appropriately.
Depending on what work the task is doing,
responding to cancellation usually means one of the following:

- Throwing an error like `CancellationError`
- Returning `nil` or an empty collection
- Returning the partially completed work

Downloading pictures could take a long time
if the pictures are large or the network is slow.
To let the user stop this work,
without waiting for all of the tasks to complete,
the tasks need to check for cancellation and stop running if they are canceled.
There are two ways a task can do this:
by calling the [`Task.checkCancellation()`][] type method,
or by reading the [`Task.isCancelled`][`Task.isCancelled` type] type property.
Calling `checkCancellation()` throws an error if the task is canceled;
a throwing task can propagate the error out of the task,
stopping all of the task's work.
This has the advantage of being simple to implement and understand.
For more flexibility, use the `isCancelled` property,
which lets you perform clean-up work as part of stopping the task,
like closing network connections and deleting temporary files.

[`Task.checkCancellation()`]: https://developer.apple.com/documentation/swift/task/checkcancellation()
[`Task.isCancelled` type]: https://developer.apple.com/documentation/swift/task/iscancelled-swift.type.property

```swift
let photos = await withTaskGroup { group in
    let photoNames = await listPhotos(inGallery: "Summer Vacation")
    for name in photoNames {
        let added = group.addTaskUnlessCancelled {
            Task.isCancelled ? nil : await downloadPhoto(named: name)
        }
        guard added else { break }
    }

    var results: [Data] = []
    for await photo in group {
        if let photo { results.append(photo) }
    }
    return results
}
```

The code above makes several changes from the previous version:

- Each task is added using the
  [`TaskGroup.addTaskUnlessCancelled(priority:operation:)`][] method,
  to avoid starting new work after cancellation.

- After each call to `addTaskUnlessCancelled(priority:operation:)`,
  the code confirms that the new child task was added.
  If the group is canceled, the value of `added` is `false` ---
  in that case, the code stops trying to download additional photos.

- Each task checks for cancellation
  before starting to download the photo.
  If it has been canceled, the task returns `nil`.

- At the end,
  the task group skips `nil` values when collecting the results.
  Handling cancellation by returning `nil`
  means the task group can return a partial result ---
  the photos that were already downloaded at the time of cancellation ---
  instead of discarding that completed work.

[`TaskGroup.addTaskUnlessCancelled(priority:operation:)`]: https://developer.apple.com/documentation/swift/taskgroup/addtaskunlesscancelled(priority:operation:)

> Note:
> To check whether a task has been canceled from outside that task,
> use the [`Task.isCancelled`][`Task.isCancelled` instance] instance property
> instead of the type property.

[`Task.isCancelled` instance]: https://developer.apple.com/documentation/swift/task/iscancelled-swift.property

For work that needs immediate notification of cancellation,
use the [`withTaskCancellationHandler(operation:onCancel:isolation:)`][] function.
For example:

[`withTaskCancellationHandler(operation:onCancel:isolation:)`]: https://developer.apple.com/documentation/swift/withtaskcancellationhandler(operation:oncancel:isolation:)

```swift
let task = Task {
    await withTaskCancellationHandler {
        // ...
    } onCancel: {
        print("Canceled!")
    }
}

// ... some time later...
task.cancel()  // Prints "Canceled!"
```

When using a cancellation handler,
task cancellation is still cooperative:
The task either runs to completion
or checks for cancellation and stops early.
Because the task is still running when the cancellation handler starts,
avoid sharing state between the task and its cancellation handler,
which could create a race condition.

<!--
  OUTLINE

  - cancellation propagates (Konrad's example below)

  ::

      let handle = Task.detached {
      await withTaskGroup(of: Bool.self) { group in
          var done = false
          while done {
          await group.addTask { Task.isCancelled } // is this child task canceled?
          done = try await group.next() ?? false
          }
      print("done!") // <1>
      }

      handle.cancel()
      // done!           <1>
-->

<!--
  Not for WWDC, but keep for future:

  task have deadlines, not timeouts --- like "now + 20 ms" ---
  a deadline is usually what you want anyhow when you think of a timeout

  - this chapter introduces the core ways you use tasks;
  for the full list what you can do,
  including the unsafe escape hatches
  and ``Task.current()`` for advanced use cases,
  see the Task API reference [link to stdlib]

  - task cancellation isn't part of the state diagram below;
  it's an independent property that can happen in any state

  [PLACEHOLDER ART]

  Task state diagram

     |
     v
  Suspended <-+
     |        |
     v        |
  Running ----+
     |
     v
  Completed

  [PLACEHOLDER ART]

  Task state diagram, including "substates"

     |
     v
  Suspended <-----+
  (Waiting) <---+ |
     |          | |
     v          | |
  Suspended     | |
  (Schedulable) / |
     |            |
     v            |
  Running --------+
     |
     v
  Completed

  .. _Concurrency_ChildTasks:

  Adding Child Tasks to a Task Group

  - awaiting ``withGroup`` means waiting for all child tasks to complete

  - a child task can't outlive its parent,
  like how ``async``-``let`` can't outlive the (implicit) parent
  which is the function scope

  - awaiting ``addTask(priority:operation:)``
  means waiting for that child task to be added,
  not waiting for that child task to finish

  - ?? maybe cover ``TaskGroup.next``
  probably nicer to use the ``for await result in someGroup`` syntax

  quote from the SE proposal --- I want to include this fact here too

  > There's no way for reference to the child task to
  > escape the scope in which the child task is created.
  > This ensures that the structure of structured concurrency is maintained.
  > It makes it easier to reason about
  > the concurrent tasks that are executing within a given scope,
  > and also enables various optimizations.
-->

<!--
  OUTLINE

  .. _Concurrency_TaskPriority:

  Setting Task Priority

  - priority values defined by ``Task.Priority`` enum

  - type property ``Task.currentPriority``

  - The exact result of setting a task's priority depends on the executor

  - TR: What's the built-in stdlib executor do?

  - Child tasks inherit the priority of their parents

  - If a high-priority task is waiting for a low-priority one,
  the low-priority one gets scheduled at high priority
  (this is known as :newTerm:`priority escalation`)

  - In addition, or instead of, setting a low priority,
  you can use ``Task.yield()`` to explicitly pass execution to the next scheduled task.
  This is a sort of cooperative multitasking for long-running work.

You can explicitly insert a suspension point
by calling the [`Task.yield()`][] method.

[`Task.yield()`]: https://developer.apple.com/documentation/swift/task/yield()

```swift
func generateSlideshow(forGallery gallery: String) async {
    let photos = await listPhotos(inGallery: gallery)
    for photo in photos {
        // ... render a few seconds of video for this photo ...
        await Task.yield()
    }
}
```

Assuming the code that renders video is synchronous,
it doesn't contain any suspension points.
The work to render video could also take a long time.
However,
you can periodically call `Task.yield()`
to explicitly add suspension points.
Structuring long-running code this way
lets Swift balance between making progress on this task,
and letting other tasks in your program make progress on their work.
-->

### Unstructured Concurrency

In addition to the structured approaches to concurrency
described in the previous sections,
Swift also supports unstructured concurrency.
Unlike tasks that are part of a task group,
an *unstructured task* doesn't have a parent task.
You have complete flexibility to manage unstructured tasks
in whatever way your program needs,
but you're also completely responsible for their correctness.

To create an unstructured task
that runs similarly to the surrounding code,
call the [`Task.init(name:priority:operation:)`][] initializer.
The new task defaults to running with
the same actor isolation, priority, and task-local state as the current task.
To create an unstructured task
that's more independent from the surrounding code,
known more specifically as a *detached task*,
call the [`Task.detached(name:priority:operation:)`][] static method.
The new task defaults to running without any actor isolation
and doesn't inherit the current task's priority or task-local state.
Both of these operations return a task that you can interact with ---
for example, to wait for its result or to cancel it.
<!-- TODO: In SE-0461 terms, Task.detached runs as an @concurrent function. -->

```swift
let newPhoto = // ... some photo data ...
let handle = Task {
    return await add(newPhoto, toGalleryNamed: "Spring Adventures")
}
let result = await handle.value
```

For more information about managing detached tasks,
see [`Task`](https://developer.apple.com/documentation/swift/task).

[`Task.init(name:priority:operation:)`]: https://developer.apple.com/documentation/swift/task/init(name:priority:operation:)-43wmk
[`Task.detached(name:priority:operation:)`]: https://developer.apple.com/documentation/swift/task/detached(name:priority:operation:)-795w1

<!--
  TODO Add some conceptual guidance about
  when to make a method do its work in a detached task
  versus making the method itself async?
  (Pull from my 2021-04-21 notes from Ben's talk rehearsal.)
-->

## Isolation

The previous sections discuss approaches for splitting up concurrent work.
That work can involve changing shared data, such as an app's UI.
If different parts of your code can modify the same data at the same time,
that risks creating a data race.
Swift protects you from data races in your code:
Whenever you read or modify a piece of data,
Swift ensures that no other code is modifying it concurrently.
This guarantee is called *data isolation*.
There are three main ways to isolate data:

1. Immutable data is always isolated.
   Because you can't modify a constant,
   there's no risk of other code modifying a constant
   at the same time you're reading it.

2. Data that's referenced by only the current task is always isolated.
   A local variable is safe to read and write
   because no code outside the task has a reference to that memory,
   so no other code can modify that data.
   In addition,
   if you capture the variable in a closure,
   Swift ensures that the closure isn't used concurrently.

3. Data that's protected by an actor is isolated
   if the code accessing that data is also isolated to the actor.
   If the current function is isolated to an actor,
   it's safe to read and write data that's protected by that actor
   because any other code that's isolated to that same actor
   must wait its turn before running.

## The Main Actor

An actor is an object that protects access to mutable data
by forcing code to take turns accessing that data.
The most important actor in many programs is the *main actor*.
In an app,
the main actor protects all of the data that's used to show the UI.
The main actor takes turns rendering the UI,
handling UI events,
and running code that you write that needs to query or update the UI.

Before you start using concurrency in your code,
everything runs on the main actor.
As you identify long-running or resource-intensive code,
you can move this work off the main actor
in a way that's still safe and correct.

> Note:
> The main actor is closely related to the main thread,
> but they're not the same thing.
> The main actor has private mutable state,
> and the main thread serializes access to that state.
> When you run code on the main actor,
> Swift executes that code on the main thread.
> Because of this connection,
> you might see these two terms used interchangeably.
> Your code interacts with the main actor;
> the main thread is a lower-level implementation detail.

<!--
TODO: Discuss the SE-0478 syntax for 'using @MainActor'

When you're writing UI code,
you often want all of it to be isolated to the main actor.
To do this, you can write `using @MainActor`
at the top of a Swift file to apply that attribute by default
to all the code in the file.
If there's a specific function or property
that you want to exclude from `using @MainActor`,
you can use the `nonisolated` modifier on that declaration
to override the default.
Modules can be configured to be built using `using @MainActor` by default.
This can be overridden on a per-file basis
by writing `using nonisolated` at the top of a file.
-->

There are several ways to run work on the main actor.
To ensure a function always runs on the main actor,
mark it with the `@MainActor` attribute:

```swift
@MainActor
func show(_: Data) {
    // ... UI code to display the photo ...
}
```

In the code above,
the `@MainActor` attribute on the `show(_:)` function
requires this function to run only on the main actor.
Within other code that's running on the main actor,
you can call `show(_:)` as a synchronous function.
However,
to call `show(_:)` from code that isn't running on the main actor,
you have to include `await` and call it as an asynchronous function
because switching to the main actor introduces a potential suspension point.
For example:

```swift
func downloadAndShowPhoto(named name: String) async {
    let photo = await downloadPhoto(named: name)
    await show(photo)
}
```

In the code above,
both the `downloadPhoto(named:)` and `show(_:)` functions
might suspend when you call them.
This code also shows a common pattern:
Perform long-running and CPU-intensive work in the background,
and then switch to the main actor to update the UI.
Because the `downloadAndShowPhoto(named:)` function isn't on the main actor,
the work in `downloadPhoto(named:)` also doesn't run on the main actor.
Only the work in `show(_:)` to update the UI runs on the main actor,
because that function is marked with the `@MainActor` attribute.
<!-- TODO
When updating for SE-0461,
this is a good place to note
that downloadPhoto(named:) runs
on whatever actor you were on when you called it.
-->

To ensure that a closure runs on the main actor,
you write `@MainActor` before the capture list and before `in`,
at the start of the closure.

```swift
let photo = await downloadPhoto(named: "Trees at Sunrise")
Task { @MainActor in
    show(photo)
}
```

The code above is similar to
`downloadAndShowPhoto(named:)` from the previous code listing,
but the code in this example doesn't wait for the UI update.
You can also write `@MainActor` on a structure, class, or enumeration
to ensure all of its methods and all access to its properties
run on the main actor:

```swift
@MainActor
struct PhotoGallery {
    var photoNames: [String]
    func drawUI() { /* ... other UI code ... */ }
}
```

The `PhotoGallery` structure in the code above
draws the photos on screen,
using the names from its `photoNames` property
to determine which photos to display.
Because `photoNames` affects the UI,
code that changes it needs to run on the main actor
to serialize that access.

When you're building on top of a framework,
that framework's protocols and base classes
are typically already marked `@MainActor`,
so you don't usually write `@MainActor` on your own types in that case.
Here's a simplified example:

```swift
@MainActor
protocol View { /* ... */ }

// Implicitly @MainActor
struct PhotoGalleryView: View { /* ... */ }
```

In the code above,
a framework like SwiftUI defines the `View` protocol.
By writing `@MainActor` on the protocol declaration,
types like `PhotoGalleryView` that conform to the protocol
are also implicitly marked `@MainActor`.
You'd see the same behavior if `View` were a base class
and `PhotoGalleryView` were a subclass ---
the subclass would be implicitly marked `@MainActor`.

In the examples above,
`PhotoGallery` protects the entire structure on the main actor.
For more fine-grained control,
you can write `@MainActor` on just the properties or methods
that need to be accessed or run on the main thread:

```swift
struct PhotoGallery {
    @MainActor var photoNames: [String]
    var hasCachedPhotos = false

    @MainActor func drawUI() { /* ... UI code ... */ }
    func cachePhotos() { /* ... networking code ... */ }
}
```

In the version of `PhotoGallery` above,
the `drawUI()` method draws the gallery's pictures on screen,
so it needs to be isolated to the main actor.
The `photoNames` property doesn't directly create the UI,
but it does store state that the `drawUI()` function uses to draw the UI,
so this property also needs to be accessed only on the main actor.
In contrast,
changes to the `hasCachedPhotos` property
don't interact with the UI,
and the `cachePhotos()` method doesn't access any state
that requires running it on the main actor.
So these aren't marked with `@MainActor`.

As with the earlier examples,
if you're using a framework to build your UI,
the property wrappers from that framework
probably already mark your UI state properties as `@MainActor`.
When defining a property wrapper,
if its `wrappedValue` property is marked `@MainActor`,
then any property you apply that property wrapper to
is also implicitly marked `@MainActor`.

## Actors

Swift provides the main actor for you ---
you can also define your own actors.
Actors let you safely share information between concurrent code.

Like classes, actors are reference types,
so the comparison of value types and reference types
in <doc:ClassesAndStructures#Classes-Are-Reference-Types>
applies to actors as well as classes.
Unlike classes,
actors allow only one task to access their mutable state at a time,
which makes it safe for code in multiple tasks
to interact with the same instance of an actor.
For example, here's an actor that records temperatures:

```swift
actor TemperatureLogger {
    let label: String
    var measurements: [Int]
    private(set) var max: Int

    init(label: String, measurement: Int) {
        self.label = label
        self.measurements = [measurement]
        self.max = measurement
    }
}
```

<!--
  - test: `actors, actors-implicitly-sendable`

  ```swifttest
  -> actor TemperatureLogger {
         let label: String
         var measurements: [Int]
         private(set) var max: Int

         init(label: String, measurement: Int) {
             self.label = label
             self.measurements = [measurement]
             self.max = measurement
         }
     }
  ```
-->

You introduce an actor with the `actor` keyword,
followed by its definition in a pair of braces.
The `TemperatureLogger` actor has properties
that other code outside the actor can access,
and restricts the `max` property so only code inside the actor
can update the maximum value.

You create an instance of an actor
using the same initializer syntax as structures and classes.
When you access a property or method of an actor,
you use `await` to mark the potential suspension point.
For example:

```swift
let logger = TemperatureLogger(label: "Outdoors", measurement: 25)
print(await logger.max)
// Prints "25".
```

In this example,
accessing `logger.max` is a possible suspension point.
Because the actor allows only one task at a time to access its mutable state,
if code from another task is already interacting with the logger,
this code suspends while it waits to access the property.

In contrast,
code that's part of the actor doesn't write `await`
when accessing the actor's properties.
For example,
here's a method that updates a `TemperatureLogger` with a new temperature:

```swift
extension TemperatureLogger {
    func update(with measurement: Int) {
        measurements.append(measurement)
        if measurement > max {
            max = measurement
        }
    }
}
```

The `update(with:)` method is already running on the actor,
so it doesn't mark its access to properties like `max` with `await`.
This method also shows one of the reasons
why actors allow only one task at a time to interact with their mutable state:
Some updates to an actor's state temporarily break invariants.
The `TemperatureLogger` actor keeps track of
a list of temperatures and a maximum temperature,
and it updates the maximum temperature when you record a new measurement.
In the middle of an update,
after appending the new measurement but before updating `max`,
the temperature logger is in a temporary inconsistent state.
Preventing multiple tasks from interacting with the same instance simultaneously
prevents problems like the following sequence of events:

1. Your code calls the `update(with:)` method.
   It updates the `measurements` array first.
2. Before your code can update `max`,
   code elsewhere reads the maximum value and the array of temperatures.
3. Your code finishes its update by changing `max`.

In this case,
the code running elsewhere would read incorrect information
because its access to the actor was interleaved
in the middle of the call to `update(with:)`
while the data was temporarily invalid.
You can prevent this problem when using Swift actors
because they only allow one operation on their state at a time,
and because that code can be interrupted
only in places where `await` marks a suspension point.
Because `update(with:)` doesn't contain any suspension points,
no other code can access the data in the middle of an update.

If code outside the actor tries to access those properties directly,
like accessing a structure or class's properties,
you'll get a compile-time error.
For example:

```swift
print(logger.max)  // Error
```

Accessing `logger.max` without writing `await` fails because
the properties of an actor are part of that actor's isolated local state.
The code to access this property needs to run as part of the actor,
which is an asynchronous operation and requires writing `await`.
Swift guarantees that
only code running on an actor can access that actor's local state.
This guarantee is known as *actor isolation*.

The following aspects of the Swift concurrency model
work together to make it easier to reason about shared mutable state:

- Code in between possible suspension points runs sequentially,
  without the possibility of interruption from other concurrent code.
  However,
  multiple pieces of concurrent code can run at the same time,
  so other code could be running simultaneously.

- Code that interacts with an actor's local state
  runs only on that actor.

- An actor runs only one piece of code at a time.

Because of these guarantees,
code that doesn't include `await` and that's inside an actor
can make the updates without a risk of other places in your program
observing the temporarily invalid state.
For example,
the code below converts measured temperatures from Fahrenheit to Celsius:

```swift
extension TemperatureLogger {
    func convertFahrenheitToCelsius() {
        for i in measurements.indices {
            measurements[i] = (measurements[i] - 32) * 5 / 9
        }
    }
}
```

The code above converts the array of measurements, one at a time.
While the map operation is in progress,
some temperatures are in Fahrenheit and others are in Celsius.
However, because none of the code includes `await`,
there are no potential suspension points in this method.
The state that this method modifies belongs to the actor,
which protects it against code reading or modifying it
except when that code runs on the actor.
This means there's no way for other code
to read a list of partially converted temperatures
while unit conversion is in progress.

In addition to writing code in an actor
that protects temporary invalid state by omitting potential suspension points,
you can move that code into a synchronous method.
The `convertFahrenheitToCelsius()` method above is a synchronous method,
so it's guaranteed to *never* contain potential suspension points.
This function encapsulates the code
that temporarily makes the data model inconsistent,
and makes it easier for anyone reading the code
to recognize that no other code can run
before data consistency is restored by completing the work.
It's important that Swift doesn't switch from this code
to run code from another part of the program
during that period of time.
In the future,
if you try to add concurrent code to this function,
introducing a possible suspension point,
you'll get a compile-time error instead of introducing a bug.

## Actor Reentrancy

An actor protects its state
by allowing only one piece of code to run at a time.
However, that guarantee doesn't extend across a potential suspension point.
When a method on an actor reaches an `await`,
the actor is free to run other work ---
including another call to the same method ---
before the original call resumes.
This behavior is called *reentrancy*,
and it's the default for every actor in Swift.

Reentrancy exists so actors stay responsive.
If an actor couldn't be reentered while one of its methods was suspended,
a single slow operation would block
every other piece of code that needs to talk to that actor ---
and two actors that both await each other in this way could
[deadlock](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW11).
Allowing reentrancy avoids that:
The actor makes progress on other work
while a long-running asynchronous operation is in flight.

The tradeoff is that
you can't assume an actor's state stays the same across an `await`.
Code that reads state, awaits, and then acts on what it read
may find that the state has changed in the meantime,
because another call ran during the suspension.

![Timeline showing the ImageLoader actor's thread held by the first call to image(for:), freed at await downloadImage(for:), then reentered by a second call to the same method while the first is still suspended off-thread](reentrant-actor-timeline)

### Keeping the Actor Responsive

Reentrancy isn't only a hazard to manage ---
it's also what makes an actor usable under load.
Consider an actor that downloads and caches images:

```swift
actor ImageLoader {
    private var cache: [String: Image] = [:]

    func image(for id: String) async throws -> Image {
        if let cached = cache[id] {
            return cached
        }

        print("Downloading image \(id).")
        let image = try await downloadImage(for: id)
        cache[id] = image
        print("Cached image \(id).")
        return image
    }
}
```

Consider two calls to `image(for:)` for two *different* IDs,
made close together ---
a photo grid loading several thumbnails at once, say.
The first call checks `cache`,
finds nothing for `"1234"`,
and reaches the `await` while downloading.
Because the actor is reentrant,
the second call doesn't wait for that download to finish:
It runs immediately,
checks `cache` for the unrelated `"5678"`,
finds nothing there either,
and starts its own download.
Both downloads proceed concurrently
instead of one blocking behind the other.

Reading and writing `cache` here is always safe:
Actors allow only one task to access their mutable state at a time,
which is exactly what makes it safe
for code in multiple tasks to interact with the same actor instance.
Reentrancy only changes the order calls can interleave *between* suspension points ---
it never lets two of them access `cache` at the same instant.

If both calls reach the `await` before either finishes,
one possible interleaving prints:

```
// Prints "Downloading image 1234."
// Prints "Downloading image 5678."
// Prints "Cached image 5678."
// Prints "Cached image 1234."
```

Nothing about this order matters:
`cache["1234"]` and `cache["5678"]` are two independent entries,
so there's no shared state for the interleaving to disturb.
A non-reentrant actor would still make the second call
wait for the first one's entire download to finish,
even though the two requests share nothing ---
exactly the unnecessary serialization reentrancy exists to avoid.

That's reentrancy working as intended.
But the same mechanism can go wrong in more than one way,
once two calls *do* share state.
Consider three variations on the same theme.

### Duplicate Work

Using the same `image(for:)` from above,
consider two calls with the same `id`, made close together.
The first call checks `cache`, finds nothing,
and reaches the `await` while downloading the image.
Because the actor is reentrant,
the second call can run before the first one resumes.
It also checks `cache`, also finds nothing ---
the first call hasn't written the result yet ---
and starts a second, redundant download.
Both calls eventually write to `cache[id]`,
so the image ends up downloaded twice
and the result depends on which call finishes last.
Wasteful, but not actually wrong:
Whichever download finishes last leaves a correct image in the cache.

If both calls reach the `await` before either finishes,
one possible interleaving prints:

```
// Prints "Downloading image 1234."
// Prints "Downloading image 1234."
// Prints "Cached image 1234."
// Prints "Cached image 1234."
```

That said, "wasteful" doesn't mean harmless.
Under enough concurrent load --- a cold cache after a deploy, say ---
redundant downloads for the same `id`
can compound faster than any of them finish,
a pattern sometimes called a cache stampede or thundering herd.
No single request does anything wrong,
but the aggregate can take a system down.

![Timeline showing two reentrant calls to image(for:) with the same id both missing the cache and downloading the same image independently, overlapping in time, before each writes cache["1234"] on its own](duplicate-work-timeline)

#### Avoiding Duplicate Work

Making the actor non-reentrant isn't an option ---
Swift doesn't offer that,
and losing reentrancy would risk the deadlocks described above.
You avoid the duplicate download instead
by recording that a download is already in progress
*before* awaiting anything,
so a reentrant call finds that record
and shares the result instead of starting its own download:

```swift
actor ImageLoader {
    private enum CacheEntry {
        case inProgress(Task<Image, Error>)
        case ready(Image)
    }

    private var cache: [String: CacheEntry] = [:]

    func image(for id: String) async throws -> Image {
        if let entry = cache[id] {
            switch entry {
            case .ready(let image):
                return image
            case .inProgress(let task):
                return try await task.value
            }
        }

        let task = Task {
            print("Downloading image \(id).")
            return try await downloadImage(for: id)
        }
        cache[id] = .inProgress(task)

        do {
            let image = try await task.value
            cache[id] = .ready(image)
            return image
        } catch {
            cache[id] = nil
            throw error
        }
    }
}
```

The key change is that `cache[id] = .inProgress(task)` runs synchronously,
before the first `await`.
The first call then awaits `task.value` itself,
and once the download finishes,
updates `cache[id]` to `.ready` ---
or clears the entry if the download throws.
By the time a second call could reenter the actor,
the in-progress download is already recorded:
That call finds the `.inProgress` entry
and awaits the same task instead of starting a new one,
so it receives the same result (or the same error) as the first call.

![Timeline showing the fix: the first call to image(for:) records the download as a shared Task before awaiting it, a reentrant second call finds that same Task and awaits it too, and only one network request happens](shared-task-timeline)

Even if both calls reach the actor before either finishes,
only the first one starts a download:

```
// Prints "Downloading image 1234."
```

### A Stale Overwrite

Consider an actor that caches the latest price for a stock symbol:

```swift
actor PriceCache {
    private var cache: [String: Double] = [:]

    func price(for symbol: String) async throws -> Double {
        if let cached = cache[symbol] {
            return cached
        }

        let price = try await fetchPrice(for: symbol)
        print("Caching \(symbol) at \(price).")
        cache[symbol] = price
        return price
    }
}
```

Consider two calls to `price(for:)` for the same `symbol`,
made close together while the price is moving.
The first call checks `cache`, finds nothing,
and reaches the `await` while fetching.
Because the actor is reentrant,
the second call can run before the first one resumes:
It also checks `cache`, also finds nothing,
and starts its own fetch.
Unlike the redundant downloads in the previous example,
these two fetches aren't guaranteed to return the same value ---
the price moved between them ---
and nothing ties the order the calls write to `cache[symbol]`
to the order the requests actually went out in.

If the response to the *earlier* request
happens to arrive --- and get written ---
*after* the response to the later one,
its price overwrites the newer, correct value.
`cache[symbol]` ends up holding a stale price
indistinguishable from a current one,
and every later call to `price(for:)` returns it as if it were live.

If the two fetches resolve in the opposite order
from the requests that started them,
one possible interleaving prints:

```
// Prints "Caching ACME at 105.0."
// Prints "Caching ACME at 100.0."
```

`cache["ACME"]` now holds `100.0` --- the stale price ---
even though `105.0` was the more recent one.
There's no error and no signal that anything went wrong:
Every subsequent caller just receives a wrong number.

![Timeline showing the stale-overwrite bug: two reentrant calls to price(for: ACME) fetch concurrently, the second call's fetch resolves first and writes the correct price, and the first call's slower fetch resolves afterward and silently overwrites it with a stale price](stale-overwrite-timeline)

#### Avoiding a Stale Overwrite

You avoid the stale overwrite instead
by holding a mutex around the entire method,
so only one call to `price(for:)` --- for any symbol --- runs at a time.
Use a small actor as the lock,
instead of Swift's standard library
[`Mutex`](https://developer.apple.com/documentation/synchronization/mutex),
which was introduced by
[SE-0433](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md).
An actor is already exclusive and already `await`-aware:
Its own state --- whether it's `locked`, and who's `waiting` ---
is protected the same way any actor's state is,
and a call that finds the lock held doesn't block a thread ---
it suspends on a continuation and gives its thread back,
exactly like any other `await`:

```swift
actor AsyncMutex {
    private var locked = false
    private var waiters: [CheckedContinuation<Void, Never>] = []

    func acquire() async {
        if !locked {
            locked = true
            return
        }

        await withCheckedContinuation { (continuation: CheckedContinuation<Void, Never>) in
            waiters.append(continuation)
        }

        locked = true
    }

    nonisolated func release() {
        Task { await _release() }
    }

    private func _release() {
        if !waiters.isEmpty {
            waiters.removeFirst().resume()
            return
        }

        locked = false
    }
}
```

`acquire()` returns immediately if nothing else holds the lock.
Otherwise it suspends the caller on a continuation
and records it in `waiters`, in order.
`release()` is `nonisolated` so a `defer` can call it without an `await`;
it hands the lock to the next waiter if there is one,
or marks the lock free.

The next example shows how `AsyncMutex` works on its own:
It serializes any calls that acquire it,
regardless of what runs inside the critical section ---
the only guarantee is that at most one caller is inside it at a time.
To see exactly when a call suspends and when it resumes,
add a print statement to each branch of `acquire()` and `release()`:

```swift
actor AsyncMutex {
    private var locked = false
    private var waiters: [CheckedContinuation<Void, Never>] = []

    func acquire() async {
        if !locked {
            locked = true
            print("Acquired immediately.")
            return
        }

        print("Already locked --- suspending.")
        await withCheckedContinuation { (continuation: CheckedContinuation<Void, Never>) in
            waiters.append(continuation)
        }
        print("Resumed --- handed the lock.")

        locked = true
    }

    nonisolated func release() {
        Task { await _release() }
    }

    private func _release() {
        if !waiters.isEmpty {
            print("Waking the next waiter.")
            waiters.removeFirst().resume()
            return
        }

        print("No waiters --- freeing the lock.")
        locked = false
    }
}

let lock = AsyncMutex()

func criticalWork(_ label: String) async {
    await lock.acquire()
    defer { lock.release() }

    print("\(label) acquired the lock.")
    try? await Task.sleep(for: .milliseconds(100))
    print("\(label) released the lock.")
}

Task { await criticalWork("A") }
Task { await criticalWork("B") }
Task { await criticalWork("C") }
```

Reentrancy is exactly what makes this worth tracing.
`acquire()` is itself `async`,
so if two calls happen close together, the actor can reenter:
A later call can start running before an earlier one has returned.
It might look like both calls could slip past `if !locked`
and both print "Acquired immediately."
That never happens,
because the check and the set ---
`if !locked { locked = true; ... }` --- run synchronously,
with no `await` between them.
Actor isolation guarantees only one call's synchronous code runs at a time,
so whichever caller reaches that line first
finishes setting `locked = true` before any other call can even read it ---
every later call is guaranteed to see `locked == true`
and take the suspending branch instead.
Suppose `A` reaches `acquire()` first,
and `B` and `C` both reach the actor, in that order,
while `A` still holds the lock.
Here's the full trace:

```
// Prints "Acquired immediately."
// Prints "A acquired the lock."
// Prints "Already locked --- suspending."   (B queues)
// Prints "Already locked --- suspending."   (C queues)
// Prints "A released the lock."
// Prints "Waking the next waiter."          (B, not C --- FIFO)
// Prints "Resumed --- handed the lock."
// Prints "B acquired the lock."
// Prints "B released the lock."
// Prints "Waking the next waiter."          (C)
// Prints "Resumed --- handed the lock."
// Prints "C acquired the lock."
// Prints "C released the lock."
// Prints "No waiters --- freeing the lock."
```

`B` and `C` both suspend while `A` still holds the lock,
as their "suspending" prints show,
and neither one actually acquires the lock
until *after* whichever call currently holds it
prints "Waking the next waiter."
The
[`CheckedContinuation`](https://developer.apple.com/documentation/swift/checkedcontinuation)
is what makes that ordering possible:
A suspended call to `acquire()` stops running entirely
at `await withCheckedContinuation` ---
it doesn't poll or retry ---
and remains suspended until `_release()` calls `continuation.resume()` on it.

![Timeline showing three tasks queuing for the same AsyncMutex, with FIFO ordering deciding who gets served next when the lock is released](fair-lock-timeline)

That trace also shows something else worth knowing:
`waiters` is an ordinary array,
appended to with `waiters.append(continuation)`
and drained with `waiters.removeFirst()`,
so waiters are served strictly in the order they arrived.
`A`'s release wakes `B`, not `C` ---
even though both have been waiting the whole time ---
because `B` queued first;
`B`'s own release then wakes `C` in turn.
That ordering isn't a coincidence:
`B`'s and `C`'s checks of `if !locked`,
and the `waiters.append` that follows,
each run as one atomic, uninterruptible step under actor isolation,
so there's always a well-defined arrival order between them, never a tie.
`AsyncMutex` is a fair lock as a result ---
no waiter can be starved by a later arrival cutting in line.

> Note: `AsyncMutex` carries the same danger any manual lock does.
> `acquire()` and `release()` are two independent calls,
> and nothing ties them together ---
> the type system doesn't stop you from acquiring the lock
> and never releasing it.
> Skip the release on an early return, or on a thrown error,
> and the lock stays held forever:
> Every later call to `acquire()` suspends and never resumes.
> Always release the lock whenever the current scope exits, no matter how ---
> a `defer` placed immediately after `await lock.acquire()`
> is the simplest way to guarantee that, as `PriceCache` does below.

`PriceCache` uses one `AsyncMutex` to guard its entire method body:

```swift
actor PriceCache {
    private var cache: [String: Double] = [:]
    private let lock = AsyncMutex()

    func price(for symbol: String) async throws -> Double {
        await lock.acquire()
        defer { lock.release() }

        if let cached = cache[symbol] {
            return cached
        }

        let price = try await fetchPrice(for: symbol)
        print("Caching \(symbol) at \(price).")
        cache[symbol] = price
        return price
    }
}
```

A second call for the same symbol now waits at `acquire()`
instead of racing the first:
By the time it resumes,
`cache[symbol]` already holds the result,
so it returns immediately without fetching again.

```
// Prints "Caching ACME at 105.0."
```

The trade-off is that the lock doesn't distinguish between symbols.
A call for `"MSFT"` made while `"ACME"` is still fetching also waits ---
even though the two share nothing ---
instead of proceeding concurrently the way reentrancy would otherwise allow:

```
// Prints "Caching ACME at 105.0."
// Prints "Caching MSFT at 310.0."
```

![Timeline showing PriceCache's AsyncMutex serializing calls for unrelated symbols, since the lock guards the whole method regardless of which symbol is requested](one-lock-every-symbol)

`AsyncMutex` fixes the stale overwrite
by giving up exactly the concurrency the earlier `ImageLoader` example demonstrated ---
a reasonable trade for a lock this simple and reusable, but a real one.
Use it with the same caution as any other mutex:
Nothing here prevents two unrelated calls from queuing behind each other.

> Note: A lock isn't the only way to avoid this stale overwrite.
> The same in-progress-`Task` technique `ImageLoader` uses to avoid duplicate work
> also closes this race:
> A reentrant call for the same `symbol` finds the `Task` already recorded
> and awaits its result instead of starting a second fetch.
> That's what actually fixes it ---
> with at most one fetch in flight per symbol,
> there's only ever one price to write,
> so no second, slower response is left around to overwrite a newer one.
> That fix needs no lock at all,
> and unlike `AsyncMutex`, it leaves unrelated symbols free to fetch concurrently.
>
> `AsyncMutex` is still worth knowing,
> but not because the `Task`-recording technique falls short for a single key ---
> it closes this race just as completely.
> The difference is scope:
> A recorded `Task` only ever protects the one dictionary entry it's stored in,
> so it has nothing to say about an invariant that spans more than one entry at once.
> `AsyncMutex` protects the entire critical section regardless of which key is involved,
> which is what you reach for once the thing you're protecting isn't cleanly per-key.

> Note: The term "mutex" usually refers to a data structure
> used to synchronize concurrent access to shared state across multiple threads:
> Before touching a non-thread-safe resource,
> a thread locks the mutex,
> which blocks that thread until no other thread holds the lock.
> Once the operation finishes,
> the thread releases the lock,
> letting another thread acquire it and proceed.
> Swift's standard library
> [`Mutex`](https://developer.apple.com/documentation/synchronization/mutex)
> can't do that here, though.
> The race isn't about a single read or write of `cache[symbol]` ---
> actor isolation already makes each of those safe on its own ---
> it's about the gap between checking the cache and writing to it,
> and closing that gap means holding the lock across the `await` on `fetchPrice`.
> `Mutex`'s `withLock` closure is synchronous, not `async`,
> so the compiler rejects an `await` inside it outright:
> There's no way to hold the lock across that gap at all.
> Even without that restriction,
> holding a lock that blocks a thread across a suspension point
> is dangerous in Swift's cooperative concurrency model:
> The task holding the lock can suspend and give its thread back to a limited pool,
> while other tasks block threads waiting for a lock
> that only a *resumed* task can release.
> Enough contention exhausts every thread in the pool
> and deadlocks the whole program ---
> not just the calls actually competing for the price.

### A Double Credit

Reentrancy doesn't need a cache to cause damage.
Consider a `Player` actor in a game,
where landing the killing blow on an enemy
awards a reward to whichever attacker finishes it off:

```swift
actor Player {
    let name: String
    private(set) var health: Int

    init(name: String, health: Int) {
        self.name = name
        self.health = health
    }

    var isAlive: Bool { health > 0 }

    // Returns true when this attack was the killing blow.
    func takeDamage(_ amount: Int, from attacker: String) async -> Bool {
        guard isAlive else { return false }

        // Suspension point: hit animation, damage mitigation lookup,
        // telemetry, and so on.
        try? await Task.sleep(for: .milliseconds(200))

        health -= amount
        let landedKillingBlow = health <= 0
        if landedKillingBlow {
            print("\(attacker) landed the killing blow.")
        }
        return landedKillingBlow
    }
}
```

Consider two attacks against the same `Player`,
made close together, each individually lethal.
The first call checks `isAlive` --- true ---
and reaches the `await` while its hit is processed.
Because the actor is reentrant,
the second call runs before the first resumes:
It also checks `isAlive` against the same, still-positive `health` ---
also true, since the first call hasn't subtracted anything yet ---
and proceeds past the guard as well.
Both calls eventually subtract from `health`,
and both find it at or below zero when they check,
so both report a killing blow --- awarding the same kill twice.

If both attacks reach the `await` before either resumes,
one possible interleaving prints:

```
// Prints "Vane landed the killing blow."
// Prints "Rowan landed the killing blow."
```

Two different attackers, both credited for the same kill.

![Timeline showing two reentrant calls to takeDamage(_:from:) against the same Player --- Vane's call checks isAlive, finds it true, and suspends; Rowan's call reenters before Vane resumes, checks isAlive against the same still-positive health, and also suspends; both later resume, both subtract from health, and both report landing the killing blow](double-credit-timeline)

#### Avoiding a Double Credit

The `Player` example has nothing to track in progress
the way the caching examples above do.
You avoid the double credit instead
by committing the entire outcome synchronously, before the only `await`:

```swift
actor Player {
    let name: String
    private(set) var health: Int

    init(name: String, health: Int) {
        self.name = name
        self.health = health
    }

    var isAlive: Bool { health > 0 }

    func takeDamage(_ amount: Int, from attacker: String) async -> Bool {
        guard isAlive else { return false }

        // Commit the outcome synchronously --- there's no `await` between
        // reading `health` and writing it, so nothing can reenter in between.
        health -= amount
        let landedKillingBlow = health <= 0
        if landedKillingBlow {
            print("\(attacker) landed the killing blow.")
        }

        // Only now, with the outcome already decided, is it safe to suspend.
        try? await Task.sleep(for: .milliseconds(200))

        return landedKillingBlow
    }
}
```

Because `health -= amount` and `landedKillingBlow`
both run before the method's only `await`,
no other call can interleave between them.
By the time a second attack reaches `guard isAlive`,
`health` already reflects the first attack ---
so at most one call finds a still-positive `health` to subtract from,
and at most one reports the killing blow.

Even if two lethal attacks reach the actor close together:

```
// Prints "Vane landed the killing blow."
```

![Timeline showing two reentrant calls to takeDamage(_:from:) against the same Player --- Vane's call finds isAlive true, subtracts from health, and suspends at its only await. While she is suspended, Rowan's call reenters, but its own isAlive check now reads false, because Vane already subtracted from health --- so Rowan's call returns immediately with no credit, never reaching its own subtract or an await. Only Vane resumes and reports landing the killing blow.](double-credit-fix-timeline)

As a general rule,
don't assume that state you read before an `await` is still valid after it.
If a method needs to act on state across a suspension point,
re-check that state after the `await`,
or --- as in the examples above ---
commit to a consistent piece of state synchronously before suspending,
so any call that reenters the actor sees a coherent picture.

> Note: Reentrancy only applies at potential suspension points.
> Code that runs between two `await` expressions ---
> or that never awaits at all ---
> still runs without interruption,
> exactly as described in
> <doc:Concurrency#Defining-and-Calling-Asynchronous-Functions>.

For a video introduction to this design, see
[“Protect mutable state with Swift actors”](https://developer.apple.com/videos/play/wwdc2021/10133/)
from WWDC21.

## Global Actors

The main actor is a global singleton instance of the [`MainActor`][] type.
An actor can normally have multiple instances,
each of which provides independent isolation.
This is why you declare all of an actor's isolated data
as instance properties of that actor.
However, because `MainActor` is a singleton ---
there is only ever a single instance of this type ---
the type alone is sufficient to identify the actor,
allowing you to mark main-actor isolation using just an attribute.
This approach gives you more flexibility to organize your code
in the way that works best for you.

[`MainActor`]: https://developer.apple.com/documentation/swift/mainactor

You can define your own singleton global actors
using the `@globalActor` attribute,
as described in <doc:Attributes#globalActor>.


<!--
  OUTLINE

  Add this post-WWDC when we have a more solid story to tell around Sendable

   .. _Concurrency_ActorIsolation:

   Actor Isolation

   TODO outline impact from SE-0313 Control Over Actor Isolation
   about the 'isolated' and 'nonisolated' keywords

   - actors protect their mutable state using :newTerm:`actor isolation`
   to prevent data races
   (one actor reading data that's in an inconsistent state
   while another actor is updating/writing to that data)

   - within an actor's implementation,
   you can read and write to properties of ``self`` synchronously,
   likewise for calling methods of ``self`` or ``super``

   - method calls from outside the actor are always async,
   as is reading the value of an actor's property

   - you can't write to a property directly from outside the actor

   - The same actor method can be called multiple times, overlapping itself.
   This is sometimes referred to as *reentrant code*.
   The behavior is defined and safe... but might have unexpected results.
   However, the actor model doesn't require or guarantee
   that these overlapping calls behave correctly (that they're *idempotent*).
   Encapsulate state changes in a synchronous function
   or write them so they don't contain an ``await`` in the middle.

   - If a closure is ``@Sendable`` or ``@escaping``
   then it behaves like code outside of the actor
   because it could execute concurrently with other code that's part of the actor

   exercise the log actor, using its client API to mutate state

   ::

       let logger = TemperatureSensor(lines: [
           "Outdoor air temperature",
           "25 C",
           "24 C",
       ])
       print(await logger.getMax())

       await logger.update(with: "27 C")
       print(await logger.getMax())
-->

## Sendable Types

Tasks and actors let you divide a program
into pieces that can safely run concurrently.
Inside of a task or an instance of an actor,
the part of a program that contains mutable state,
like variables and properties,
is called a *concurrency domain*.
Some kinds of data can't be shared between concurrency domains,
because that data contains mutable state,
but it doesn't protect against overlapping access.

A type that can be shared from one concurrency domain to another
is known as a *sendable* type.
For example, it can be passed as an argument when calling an actor method
or be returned as the result of a task.
The examples earlier in this chapter didn't discuss sendability
because those examples use simple value types
that are always safe to share
for the data being passed between concurrency domains.
In contrast,
some types aren't safe to pass across concurrency domains.
For example, a class that contains mutable properties
and doesn't serialize access to those properties
can produce unpredictable and incorrect results
when you pass instances of that class between different tasks.

You mark a type as being sendable
by declaring conformance to the `Sendable` protocol.
That protocol doesn't have any code requirements,
but it does have semantic requirements that Swift enforces.
In general, there are three ways for a type to be sendable:

- The type is a value type,
  and its mutable state is made up of other sendable data ---
  for example, a structure with stored properties that are sendable
  or an enumeration with associated values that are sendable.
- The type doesn't have any mutable state,
  and its immutable state is made up of other sendable data ---
  for example, a structure or class that has only read-only properties.
- The type has code that ensures the safety of its mutable state,
  like a class that's marked `@MainActor`
  or a class that serializes access to its properties
  on a particular thread or queue.

<!--
  There's no example of the third case,
  where you serialize access to the class's members,
  because the stdlib doesn't include the locking primitives you'd need.
  Implementing it in terms of NSLock or some Darwin API
  isn't a good fit for TSPL.
  Implementing it in terms of isKnownUniquelyReferenced(_:)
  and copy-on-write is also probably too involved for TSPL.
-->

For a detailed list of the semantic requirements,
see the [`Sendable`](https://developer.apple.com/documentation/swift/sendable) protocol reference.

Some types are always sendable,
like structures that have only sendable properties
and enumerations that have only sendable associated values.
For example:

```swift
struct TemperatureReading: Sendable {
    var measurement: Int
}

extension TemperatureLogger {
    func addReading(from reading: TemperatureReading) {
        measurements.append(reading.measurement)
    }
}

let logger = TemperatureLogger(label: "Tea kettle", measurement: 85)
let reading = TemperatureReading(measurement: 45)
await logger.addReading(from: reading)
```

<!--
  - test: `actors`

  ```swifttest
  -> struct TemperatureReading: Sendable {
         var measurement: Int
     }

  -> extension TemperatureLogger {
         func addReading(from reading: TemperatureReading) {
             measurements.append(reading.measurement)
         }
     }

  -> let logger = TemperatureLogger(label: "Tea kettle", measurement: 85)
  -> let reading = TemperatureReading(measurement: 45)
  -> await logger.addReading(from: reading)
  ```
-->

Because `TemperatureReading` is a structure that has only sendable properties,
and the structure isn't marked `public` or `@usableFromInline`,
it's implicitly sendable.
Here's a version of the structure
where conformance to the `Sendable` protocol is implied:

```swift
struct TemperatureReading {
    var measurement: Int
}
```

<!--
  - test: `actors-implicitly-sendable`

  ```swifttest
  -> struct TemperatureReading {
         var measurement: Int
     }
  ```
-->

To explicitly mark a type as not being sendable,
write an unavailable conformance to `Sendable`:

```swift
struct FileDescriptor {
    let rawValue: Int
}

@available(*, unavailable)
extension FileDescriptor: Sendable {}
```

<!--
The example above is based on a Swift System API.
https://github.com/apple/swift-system/blob/main/Sources/System/FileDescriptor.swift

See also this PR that adds Sendable conformance to FileDescriptor:
https://github.com/apple/swift-system/pull/112
-->

You can also use an unavailable conformance
to suppress implicit conformance to a protocol,
as discussed in <doc:Protocols#Implicit-Conformance-to-a-Protocol>.

<!--
  LEFTOVER OUTLINE BITS

  - like classes, actors can inherit from other actors

  - actors can also inherit from ``NSObject``,
  which lets you mark them ``@objc`` and do interop stuff with them

  - every actor implicitly conforms to the ``Actor`` protocol,
  which has no requirements

  - you can use the ``Actor`` protocol to write code that's generic across actors

  - In the future, when we get distributed actors,
    the TemperatureSensor example
    might be a good example to expand when explaining them.

  ::

      while let result = try await group.next() { }
      for try await result in group { }

  how much should you have to understand threads to understand this?
  Ideally you don't have to know anything about them.

  How do you meld async-await-Task-Actor with an event driven model?
  Can you feed your user events through an async sequence or Combine
  and then use for-await-in to spin an event loop?
  I think so --- but how do you get the events *into* the async sequence?

  Probably don't cover unsafe continuations (SE-0300) in TSPL,
  but maybe link to them?
-->

<!--
This source file is part of the Swift.org open source project

Copyright (c) 2014 - 2022 Apple Inc. and the Swift project authors
Licensed under Apache License v2.0 with Runtime Library Exception

See https://swift.org/LICENSE.txt for license information
See https://swift.org/CONTRIBUTORS.txt for the list of Swift project authors
-->
