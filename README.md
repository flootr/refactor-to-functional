Refactor to Functional
======================

This article helps procedural or object oriented programmers refactor the way they think to a more functional approach using practical examples in JavaScript.

We will cover the cornerstones of functional programming such as function purity, recursion, map, reduce, curry and compose.

## Why Functional?

Functional programming can be hard to wrap your head around. Yes, that’s right you’re not the only one. At it’s worst functional programming is dense and looks mathematical but written correctly it is clear, concise and easy to debug. 

Functions are described as “pure” when they fulfil two conditions:
* They always return the same result given the same input
* They have no side effects

For a function to always return the same result it cannot make any external system calls and cannot maintain any state. Classic examples of state would be the `this` variable or other global variables. These are things your function can change, but more importantly things your function relies on that other processes might change.

Similarly functions that have no side effects must not mutate existing variables or perform calls to external systems (APIs, file system etc).

Why does this matter? Pure functions are incredibly easy to test, they take 0 or more inputs and return a value. This means no mocking of dependencies or setting test harness to get the system in the correct state. Just simple input / output.

At the heart of functional programming is a belief that data is just data and the program’s core responsibility is to transform and manipulate that data.

## Map

Let’s say we have an array of numbers and we want to get the square root of each of them. That’s pretty easy right?

```javascript
function sqrtAll (values) {
    for (var i in values) {
        values[i] = Math.sqrt(values[i]);
    }

    return values;
}
```

This function mutates the state of the values array and returns the result, this has the potential to cause side effects and may lead to problems. It’s safer to return a new set of results. Using map we can return a new array of sqrt’d values. Map applies a given function over an array.

```javascript
function sqrtAll (values) {
    return map(Math.sqrt, values);
}
```

Not only is the less lines of code, it is “pure” and produces no side effects, but we can still do better.

##Curry

Functional currying is a way of partially applying a function. If that doesn’t help, think of it as a way of creating a new function based off an existing function with some arguments already supplied. Take this add function:

```javascript
function add (x, y) {
    return x + y;
}
```

We can curry it to make a new function called add5:

```javascript
var add5 = curry(add, 5);
add5(2); // Gives 7
```

On the surface of it this may seem fairly pointless, but it’s actually very powerful. Take our earlier example of sqrtAll, we can now define that as:

```javascript
var sqrtAll = curry(map, Math.sqrt);
```

Here we are saying take the map function and preload it with the Math.sqrt so all that’s left to do is provide the array of values:

```javascript
sqrtAll([9,64]); // [3, 8]
```

## Recursion

The join function takes an array of strings and joins them together with a separator. Here is a typical implementation:

```javascript
function join (values, separator) {
    var string = values.shift();

    for (var i in values) {
        string += separator + values[i];
    }

    return string;
}
```

We can also implement this using recursion:

```javascript
function join (separator, values) {
    return values.length === 1
            ? values[0]
            : values[0] + separator + join(separator, values.slice(1));

}
```

This function deals with two scenarios: it has reached the end of this list, in which case it just returns the value or it is not at the end and returns the next value + separator + the result of join with the rest of the array. This function will keep calling itself removing one from the array each time until it reaches the end.

## Reduce

Reduce is a special case of recursion where an a data structure (e.g. an array)  is reduced down to single value. For this example we will look at sum.

```javascript
function sum (values) {
    var total = 0;

    for (var i in values) {
        total += values[i];
    }

    return total;
}

var total = sum([1, 2, 3]); // 6
```

The procedural version of this code looks much the same as our join function. The reduce function handles our recursion so all we need to do is the addition.

```javascript
function add (value, accumulator) {
    return value + result;
}

var total = reduce(add, [1, 2, 3]); // 6
```

Or when combined with curry:

```javascript
var sum = curry(reduce, add); 
var total = sum([1, 2, 3]); // 6
```

In a normal reduce implementation it will be expecting 2 or 3 arguments; the function, the array and an optional accumulator (starting point). The function will be recursively called for each item in the array, each time receiving the value and current accumulated result.

## Compose

Using compose we can chain together any number of functions to return a result. This is great if you need to perform a number of actions on the same dataset. Let’s take our earlier examples of sqrtAll and sum. If we want to get the total of the sqrt of each item in an array we could write:

```javascript
var squareRoots = sqrtAll([9, 64]);
var total = sum(squareRoots);
```

But it is much more concise to use compose:

```javascript
var total = compose(sum, sqrtAll, [9, 64]); 
```

This code reads very nicely, albeit from right to left. With this array, square root each item and sum the result.

## Real World Examples

All the examples so far have been fairly artificial to keep them short and to the point. Now let’s go over some more real world scenarios.

### jQuery Callbacks

jQuery ajax requests take callbacks for error and success handling. Quite often we will want to provide some UI feedback based on the request result. Let’s say we want to display a dialog box with either an error message or a success message we could write it like this:

```javascript
// let’s start a loading icon 
startLoadingSpinner();

$.get('/api/endpoint', {
    success: displaySuccess,
    error: displayError
}); 

function displaySuccess() {
    stopLoadingSpinner();
    showDialog('Success!');
}

function displayError() {
    stopLoadingSpinner();
    showDialog('Oh no, error!');
}
```

Obviously the displayError and displaySuccess functions are very similar and need to be refactored. Now we’ll use curry to tidy things up a bit:

```javascript
startLoadingSpinner();

$.get('/api/endpoint', {
    success: curry(requestComplete, 'Success!'),
    error: curry(requestComplete, 'Oh no, error!')
}); 

function requestComplete(message) {
    stopLoadingSpinner();
    showDialog(message);
}
```

### Event Stream

In this scenario we have a list of events containing a type (create, edit, delete) and a date (yyyy-mm-dd). We want to roll up all the events of the same type on the same day into a single event.

We can make a composite key of the date and type to remove duplicates.

```javascript
function reduceEvents(events) {
    var eventStream = {};

    for (var i in events) {
        var event = events[i];
        eventStream[event.type + event.date] = event;
    }

    return events;
}
```

Pretty easy right? Now let’s add a couple of complications:
* The events are ordered by date descending (most recent first) and the current function keeps oldest event. We want to display the most recent event.
* We want to limit the final result to 10

Now we have:

```javascript
function reduceEvents (events) {
    var eventStream = {};

    events.reverse();

    for (var i in events) {
        var event = events[i];
        eventStream[event.type + event.date] = event;
    }

    events.reverse();

    return events.slice(0, 10);
}
```

This code is not complicated but to someone reading it for the first time it is not immediately obvious what is happening until they read the entire function. Using a few functional techniques we can make it more conscise and readable:

```javascript
function reduceEvents (events) {
    var reduceEvent = curry(reduce, reduceEvent);
    var first10 = curry(slice, 10); 
    
    return compose(
       first10,
       reverse,
       reduceEvent,
       reverse,
       events
    );
}

function reduceEvent (event, accumulator) {
    accumulator[event.type + event.date] = event;

    return accumulator;
}
```

What’s nice about this is that the reduceEvents function is much more descriptive. Rather than reading through the code we can just see what it is doing: with the events, do a reverse, reduce them, reverse them back and take the first 10. If you don’t like reading from right to left some libraries provide a chain function so it could be written as:

```javascript
return chain(events)
        .then(reverse)
        .then(reduceEvent)
        .then(reverse)
        .then(first10)
        .values();
```

Speaking of libraries the examples here will work with [fun-js](https://github.com/briansorahan/fun-js) or [scoreunder](https://github.com/loop-recur/scoreunder) (if you add _. to the function calls). [Underscore](http://underscore.org) is a great library but the ordering of parameters makes it [difficult to curry and compose](http://www.youtube.com/watch?v=m3svKOdZijA).
