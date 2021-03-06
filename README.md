# Tc-39 proposal research: Promise.all() match iterator

## Proposal

The `Promise.all(iterable)` takes `iterable`, an object implementing the [iterable protocol](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Iteration_protocols#iterable). The subsequent fulfillment callback receives `result`, which seems to be an `Array` built using `for..of` over `iterable`. This proposal, suggests the implementation should provide *some way* to make the output `result` the same type as the input `iterable`, for [builtin iterables](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Iteration_protocols#Builtin_iterables).

## Goal of this project

This repository makes a suggestion to the implementation of [Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all). I will use it to try and understand the pros and cons of the proposal, and decide what, if any further action is warranted.

## Background

When iterating over a `Map` using `for ... of` you get an `[key. value]` array (destructured in the example below). Plus there are [other ways to iterate](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map#Iterating_Maps_with_for..of)

```javascript
var myMap = new Map();
myMap.set(0, "zero");
myMap.set(1, "one");
for (var [key, value] of myMap) {
  console.log(key + " = " + value);
}
// Will show 2 logs; first with "0 = zero" and second with "1 = one"
```

## Example introduction

`Promise.all` can take any object that implements the [iterable protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols), such as the built-in `Map` and `Array` types. However the fulfillment callback of `promise.all` is always passed an `Array` of ouputs from the iteration. It is my feeling this type change from `<Iterable>` (input) to `Array` of  (output) is an unnecesary and unexpected mutation.

For instance, given an asynchronous _promisified_ random number generator:

```javascript
// src/example.js::lines:16-36
    var now = function () {
        return (new Date()).valueOf();
    };

    // Async random generator:
    var extraRandom = function (callback) {
        let start = now();
        let doCallback = function () {
            callback(Math.random() * 10);
        };
        setTimeout(function() {
            doCallback( now() - start );
        },  Math.random() * 10);
    };

    // Promises a random number, at a randomly later time:
    var promiseExtraRandom = function () {
        return new Promise(function (done) {
            extraRandom(done);
        });
    };
```

A simple game can be created by loading a pair of promises into an iterator. The iterator could be passed to `Promise.all`, and the subsequent result can be examined to find the highest random number a.k.a the "winner".

## Current behavior: Array in, Array out

Currently, `Promise.all` takes any iterable, like an `Array`, and in the case of an `Array` input, it provides an `Array` to the subsequent `then()` call:

```javascript
// /src/example.js::lines:42-49
        dataStore = [];
        dataStore.push(promiseExtraRandom());
        dataStore.push(promiseExtraRandom());
        dataStoreDone = Promise.all(dataStore).then( function (result) {
            var winner = ( result[0] > result[1] ) ? '1' : '2';
            console.log( '>> input:', Object.getPrototypeOf(dataStore), '>> output:', Object.getPrototypeOf(result));
            console.log( '>> Player at index "' + winner + '" won\n\n');
        });

// ...

        // Here the dataStore is treated like an Object..
        dataStore.forEach(function (val, key) {
            console.log('Brave "' + key + '" has entered the game.');
        });
```

Output:

```
Brave "0" has entered the game.
Brave "1" has entered the game.
>> input: [] >> output: []
>> Player at index "2" won
```

## Current behavior: Map in, Array out

Now when `Promise.all` is given a different iterator, like a `Map` the `result` argument of `then()` is still an `Array`, destroying the key-to-value representation:

```javascript
// src/example.js::lines:54-62
        dataStore = new Map();
        dataStore.set('Ryu', promiseExtraRandom());
        dataStore.set('Ken', promiseExtraRandom());
        dataStoreDone = Promise.all(dataStore).then( function (result) {
            var [ ryu, ken ] = result;
            var winner = ( ryu > ken ) ? 'Ryu (implied by result at index 1)' : 'Ken (implied by result at index 2)';
            console.log( '>> input:', Object.getPrototypeOf(dataStore), '>> output:', Object.getPrototypeOf(result));
            console.log( '>> "' + winner + '" won\n\n');
        });

// ...

        // Here the dataStore is treated like an Object..
        dataStore.forEach(function (val, key) {
            console.log('Brave "' + key + '" has entered the game.');
        });
```


Output:

```
Brave "Ryu" has entered the game.
Brave "Ken" has entered the game.
>> input: Map {} >> output: []
>> "Ryu (implied by result at index 1)" won
```


Notice I attempt to recreate the key-to-value relationship with the de-structured assignment, which works well for two players but not _N_ players. `Promise.all` gives a result that is an array of built by iterating over the `Map` using `for...of`, rather than providing the `Map` itself. *The merits of this proposal largely hinges upon the judgement on what is lost in this transformation.*

## Proposed behavior: Map in, Map out

Now when the promises in `dataStore` are fulfilled and the callback is invoked, I'd like (and had expected) to interact with the `result`, as if it was the `dataStore`. As the input was an iterable `Map`, I'd like to be able to iterate over the value _and keys_ :

```javascript
// src/example.js::lines:68-80
        dataStore = new Map();
        dataStore.set('Ryu', promiseExtraRandom());
        dataStore.set('Ken', promiseExtraRandom());
        dataStoreDone = promiseAllMatchIterator(dataStore).then( function (result) {
            var winner;
            result.forEach(function (val, key) {
                if ( !winner || val > winner.val ) {
                    winner = {val, key}; 
                }
            });
            console.log( '>> input:', Object.getPrototypeOf(dataStore), '>> output:', Object.getPrototypeOf(result));
            console.log( '>> The hero, "' + winner.key + '" won\n\n' );
        });

// ...

        // Here the dataStore is treated like an Object..
        dataStore.forEach(function (val, key) {
            console.log('Brave "' + key + '" has entered the game.');
        });
```

Output:

```
Brave "Ryu" has entered the game.
Brave "Ken" has entered the game.
>> input: Map {} >> output: Map {}
>> The hero, "Ryu" won
```

## Proposal partial implementation

```javascript
// src/promise-all-match-iterable.js::lines:12-36
export default function (iterator) {
    let hasDefaultIterableBehavior = function (iterator) {
        return true; // Not implemented yet
    };
    return Promise.all(iterator).then( function (result) {
        var payload = result;
        if (hasDefaultIterableBehavior(iterator)) {
            if (Object.getPrototypeOf(iterator) === Map.prototype) {
                var index = 0,
                    keys = iterator.keys(),
                    payload = new Map();
                for (let k of keys) {
                    let v = result[ index++ ];
                    payload.set( k, v );
                }
            } else if (Object.getPrototypeOf(iterator) === TypedArray.prototype) {
                throw new Error("promise-all-match-iterable - TypedArray not handled yet");
            } else if (Object.getPrototypeOf(iterator) === Set.prototype) {
                throw new Error("promise-all-match-iterable - Set not handled yet");
            }

        }
        return payload;
    });
};
```

## Complications, and overview of solution

Because the iterator protocol could be implemented in an arbitrary (a.k.a unexpected) way by an object, it cannot be anticipated. However, for in-built types, the behavior is known. Thus, given an iteractor input, `Promise.all` could return a result of the same type (with the same characterstics).

## Next steps

* Extend proposed implementation to work with other builtin iterables
* Look at overwriting the default iterable behavior of builtin iterables

## Feedback

As you wish, but I'd suggest [leaving an issue](https://github.com/AshCoolman/tc-39-suggestion-promise-result-type/issues)

## About this repository

Install:

```shell
npm i
```


Run `src/example.js`:

```shell
npm test
```


Re-generate this readme from `src/assets/README.src.md`:

```shell
npm run build-readme
```

