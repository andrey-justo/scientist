# Scientist [![Build Status](https://travis-ci.org/spoptchev/scientist.svg?branch=master)](https://travis-ci.org/spoptchev/scientist)

A kotlin library for carefully refactoring critical paths in your application.

This library is inspired by the ruby gem [scientist](https://github.com/github/scientist).

## How do I science?

Let's pretend you're changing the way you handle permissions in a large web app. Tests can help guide your refactoring, but you really want to compare the current and refactored behaviors under load.


```kotlin
fun isAllowed(user): Boolean = scientist<Boolean, Unit>() conduct {
    experiment { "widget-permissions" }
    control { user.isAllowedOldWay() }
    candidate { user.isAllowedNewWay() }
}
```

Wrap a `control` block around the code's original behavior, and wrap `candidate` around the new behavior. When conducting the experiment `conduct` will always return whatever the `control` block returns, but it does a bunch of stuff behind the scenes:

* It decides whether or not to run the `candidate` block,
* Randomizes the order in which `control` and `candidate` blocks are run,
* Measures the durations of all behaviors,
* Compares the result of `candidate` to the result of `control`,
* Swallows (but records) any exceptions raised in the `candidate` block, and
* Publishes all this information.

## Scientist and Experiment

Compared to other scientist libraries this library separates the concepts of a scientist and the experiment.
Which in turn gives you more freedom and flexibility to compose and reuse scientists and experiments especially with dependency injection frameworks.

```kotlin
val scientist = scientist<Boolean, Unit>()
val experiment = experiment<Boolean, Unit() {
    control { true }
}

val result = scientist conduct experiment
```

### Setting up the scientist

The scientist is responsible for setting up the environment of an experiment and conducting it.

#### Publishing results

The examples above will run, but they're not really *doing* anything. The `candidate` blocks run every time and none of the results get published. Add a publisher to control the result reporting:

```kotlin
val scientist = scientist<Boolean, Unit> {
    publisher { result -> logger.info(result) }
}
```

You can also extend the publisher `typealias` which then can be used as a parameter of the publisher block:

```kotlin
val logger = loggerFactory.call()

class LoggingPublisher(val logger: Logger) : Publisher<Boolean, Unit> {
    override fun invoke(result: Result<Boolean, Unit>) {
        logger.info(result)
    }
}

val loggingPublisher = LoggingPublisher(logger)

val scientist = scientist<Boolean, Unit> {
    publisher(loggingPublisher)
}
```

#### Controlling matches

Scientist compares if control and candidate values have matched by using `==`. To override this behavior, use `match` to define how to compare observed values instead:

```kotlin
val scientist = scientist<Boolean, Unit> {
    match { candidate, control -> candidate != control }
}
```

`candidate` and `control` are both of type `Outcome` (a sealed class) which either can be `Success` or `Failure`, so you can easily reason about them. As an example take a look at the default implementation:

```kotlin
class DefaultMatcher<in T> : Matcher<T> {
    override fun invoke(candidate: Outcome<T>, control: Outcome<T>): Boolean = when(candidate) {
        is Success -> when(control) {
            is Success -> candidate == control
            is Failure -> false
        }
        is Failure -> when(control) {
            is Success -> false
            is Failure -> candidate.errorMessage == control.errorMessage
        }
    }
}
```

A `Success` outcome contains the value that has been evaluated. A `Failure` outcome contains the exception that was caught while evaluating a `control` or `candidate` statement.

#### Adding context

To provide additional data to the scientist `Result` and `Experiments` you can use the `context` block to add a context provider:

```kotlin
val scientist = scientist<Boolean, Map<String, Boolean>> {
    context { mapOf("yes" to true, "no" to false) }
}
```

The context is evaluated lazily and is exposed to the publishable `Result` by evaluating `val context = result.contextProvider()` and in the experiments `conductibleIf` lambda that will be described a further down the page.

#### Ignoring mismatches

During the early stages of an experiment, it's possible that some of your code will always generate a mismatch for reasons you know and understand but haven't yet fixed. Instead of these known cases always showing up as mismatches in your metrics or analysis, you can tell the scientist whether or not to ignore a mismatch using the `ignore` block. You may include more than one block if needed:

```
val scientist = scientist<Boolean, Map<String, Boolean>> {
    ignore { candidate, control -> candidate.isFailure() }
}
```

Like in `match` candidate and control are of type `Outcome`.

#### Putting it all together

```kotlin
val scientist = scientist<Boolean, Map<String, Boolean>> {
    publisher { result -> logger.info(result) }
    match { candidate, control -> candidate != control }
    context { mapOf("yes" to true, "no" to false) }
    ignore { candidate, control -> candidate.isFailure() }
}
```

### Setting up an experiment

