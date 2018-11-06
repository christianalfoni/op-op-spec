# op-op-spec
The op-op specification. A simple straight forward approach to functional programming

## What is it?

Op-op is just a signature that allows infinite async composition of logic with error handling.

**The signature**
```js
(err, value, next, final = next) => {}
```

**The type signature**
```ts
type Operator<Input, Output = Input> = (
  err: Error,
  value: Input,
  next: (err: Error, value: Output),
  final: (err: Error, value: Output)
) => void
```

## Understand how it works

The simplest implementation of op-op is to create a function with the signature:

```js
const doThis = (err, value, next, final = next) => {
  if (err) next(err)
  else next(null, value)
}
```

Which you could call like this:

```js
doThis(null, "foo", console.log) // null, "foo"
```

This function is now an **operator** and it is the core building block of op-op. Let us use this signature to create a more powerful abstraction. Let us name it **pipe**:

```js
const pipe = (...initialOperators) => (err, value, next, final = next) => {
  if (err) next(err)
  else {
    const runNextOperator = (operators, operatorError, operatorValue) {
      if (operatorError) next(operatorError)
      else if (operators.length) {
        const [nextOperator, ...remainingOperators]  = operators
        nextOperator(null, operatorValue, runNextOperator.bind(null, remainingOperators), final) 
      }
      else next(null, operatorValue)
    }
    runNextOperator(initialOperators, null, val)
  }
}
```

With this operator we are now able to run multiple **doThis**:

```js
const manyDoThis = pipe(
  doThis, // null, "foo"
  doThis, // null, "foo"
  doThis // null, "foo"
)

manyDoThis(null, "foo", console.log) // null, "foo"
```

But we are still not really doing anything interesting. Let us introduce our first userful operator, **map**:

```js
const map = operation => (err, value, next) => {
  if (err) next(err)
  else next(null, operation(value))
}
```

Now we can do this:

```js
const doThis = pipe(
  map(value => value.toUpperCase()),
  map(value => value + "!!!")
)

doThis(null, "hello world", console.log) // null, "HELLO WORLD!!!"
```

Or let us be super declarative here:

```js
const toUpperCase = map(value => value.toUpperCase())
const shout = map(value => value + "!!!")

const doThis = pipe(
  toUpperCase,
  shout
)

doThis(null, "hello world", console.log) // null, "HELLO WORLD!!!"
```

Again, not extremly helpful, so let us imagine:

```js
const search = action(
  setQuery,
  filterValidQuery,
  debounce(200),
  queryServer,
  handleQueryError,
  setQueryResult
)
```

Now we are starting to see how effective you can be at expressing complex logic. Exactly how these operators would be implemented depends heavily on where to use them, which is why this is only a spec. Though continue further for more inspiration on creating operators and maybe you find a specific use case where you want to expose a functional API.

## Example operators

### Wait

```js
const wait = ms => (err, value, next) => {
  if (err) next(err)
  else setTimeout(() => next(null, value), ms)
}
```

### Filter
```js
const filter = operation => (err, value, next, final) => {
  if (err) next(err)
  else if (operation(value)) next(null, value)
  else final(null, value)
}
```

### Debounce

```js
const debounce = ms => {
  let timeout
  let previousFinal
  return (err, value, next, final) => {
    if (err) {
      return next(err)
    }
    if (timeout) {
      clearTimeout(timeout)
      previousFinal(null, value)
    }
    previousFinal = final
    timeout = setTimeout(() => {
      timeout = null
      next(null, value)
    }, ms)
  }
}
```

### Pipe (async)

This operator transparently handles values that are promises. Meaning that for example **map** could return a promise or be an async function, and it would wait for it to resolve before moving on to next operator.

```js
const pipe = (...initialOperators) => (err, value, next, final) => {
  if (err) next(err)
  else {
    const runNextOperator = (operators, operatorError, operatorValue) => {
      if (operatorError) next(operatorError)
      else if (operators.length) {
        const [nextOperator, ...remainingOperators] = operators
        const run = (runErr, runValue) =>
          nextOperator(runErr, runValue, runNextOperator.bind(null, remainingOperators), final || next)

        if (operatorValue instanceof Promise) {
          operatorValue
            .then(promiseValue => run(null, promiseValue))
            .catch(promiseError => run(promiseError, operatorValue))
        } else {
          try {
            run(null, operatorValue)
          } catch (operatorError) {
            run(operatorError, operatorValue)
          }
        }
      }
      else next(null, operatorValue)
    }
    runNextOperator(initialOperators, null, value)
  }
}
```
