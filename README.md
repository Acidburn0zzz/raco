# caco

Generator based control flow that supports both callbacks and promises.

[![Build Status](https://travis-ci.org/cshum/caco.svg?branch=master)](https://travis-ci.org/cshum/caco)

```bash
npm install caco
```

Many existing flow-control libraries such as [co](https://github.com/tj/co), assume promises to be the lowest denominator of async handling.
Callback functions require promisify to be compatible, which creates unnecessary complication. 

In caco, both callbacks and promises are yieldable.
Resulting function can also be used by both callbacks and promises.
This enables a powerful control flow while maintaining simplicity.

#### var fn = caco(fn*, [mapper])
#### caco(obj, [mapper])

```js
var caco = require('caco')

var fn = caco(function * (next) {
  try {
    yield Promise.reject('boom') // yield promise reject throws error
  } catch (err) {
    console.log(err) // 'boom'
  }

  var foo = yield Promise.resolve('bar') // yield promise

  yield setTimeout(next, 1000) // yield callback using 'next' argument, delay 1 second

  // yield callback of form next(err, data): return data, throw if err exists
  var data = yield fs.readFile('./foo/bar', next) 

  return data
})

// Use with callback
fn(function (err, res) { })

// Use with promise
fn().then(...).catch(...)
```

Wraps generator function properties of object:

```js
function App () {
  ...
}
App.prototype.fn = function * (arg, next) {
  ...
}

// wrap prototype object
caco(App.prototype)

var app = new App()

fn(function (err, res) { ... })
fn().then(...).catch(...)
```

## Yieldables

Yieldable callback works by supplying an additional `next` argument. Yielding non-yieldable value pauses the current generator. 
Until `next(err, val)` being invoked by callback, 
where `val` passes back to yielded value, or `throw` if `err` exists.

By default, the following objects are considered yieldable:
* `Promise`
* `Observable`
* `Generator`

Caco also accepts a yield mapper callback function, 
so that one can yield pretty much anything.

```js
function mapper (val, cb) {
  // map array to Promise.all
  if (Array.isArray(val)) {
    Promise.all(val).then(function (res) {
      cb(null, res)
    }, cb)
    return true // acknowledge yieldable
  }

  // Anything can be mapped!
  if (val === 689) {
    cb(new Error('DLLM'))
    return true
  }
}

caco(function * () {
  console.log(yield [
    Promise.resolve(1),
    Promise.resolve(2),
    3
  ]) // [1, 2, 3]

  // yield 689 throws error
  try {
    yield 689
  } catch (err) {
    console.log(err.message) // 'DLLM'
  }
}, mapper)(function (err, res) {
  // handle error or return
})

```

## Aggregated Yield

Multiple results can be aggregated in one `yield` by using `Promise.all` or [callback-all](https://github.com/cshum/callback-all).

```js
var caco = require('caco')
var cball = require('callback-all')

caco(function * (next) {
  // Promise.all
  var promises = [
    asyncFn1() // foo
    asyncFn2() // bar
    asyncFn3() // hello
    asyncFn4() // world
  ]
  console.log(yield Promise.all(promises)) // ['foo', 'bar', 'hello', 'world']

  // callback-all
  var all = cball()
  asyncFn1(all()) // foo
  asyncFn2(all()) // bar
  asyncFn3(all()) // hello
  asyncFn4(all()) // world
  console.log(yield all(next)) // ['foo', 'bar', 'hello', 'world']

})(function (err, res) {
 // handle error or return
})
```

## License

MIT
