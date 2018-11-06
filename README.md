# op-op-spec
The op-op specification. A simple straight forward approach to functional programming with type support

## What is it?

Op-op is just a signature that allows infinite async composition of logic with error handling.

**The signature**
```js
function operator (err, value, next, final = next) {}
```

**The type signature**
```ts
type Operator<Input, Output> = (
  err: Error,
  value: Input,
  next: (err: Error, value?: Output) => void,
  final: (err: Error, value?: Output) => void
) => void
```

## Understand how it works

The simplest implementation of op-op is to create a function with the signature:

```js
function operator (err, value, next, final = next) {
  if (err) next(err)
  else next(null, value)
}
```

Which you could call like this:

```js
operator(null, "foo", console.log) // null, "foo"
```

This function is now an **operator** and it is the core building block of op-op. Let us use this signature to create a more powerful abstraction. Let us name it **pipe**:

```js
function pipe (...initialOperators) {
  return (err, value, next, final = next) => {
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
      runNextOperator(initialOperators, null, value)
    }
  }
}
```

With this operator we are now able to run multiple operators:

```js
const operators = pipe(
  operator, // null, "foo"
  operator, // null, "foo"
  operator // null, "foo"
)

operators(null, "foo", console.log) // null, "foo"
```

But we are still not really doing anything interesting. Let us introduce our first useful operator, **map**:

```js
function map (operation) {
  return (err, value, next, final = next) => {
    if (err) next(err)
    else next(null, operation(value))
  }
}
```

Now we can do this:

```js
const operator = pipe(
  map(value => value.toUpperCase()),
  map(value => value + "!!!")
)

operator(null, "hello world", console.log) // null, "HELLO WORLD!!!"
```

Or let us be super declarative here:

```js
const toUpperCase = map(value => value.toUpperCase())
const shout = map(value => value + "!!!")

const operator = pipe(
  toUpperCase,
  shout
)

operator(null, "hello world", console.log) // null, "HELLO WORLD!!!"
```

We would like to make this operator callable, like a plain function passing a value:

```js
const toFunc = (...operators) => {
  const run = pipe(...operators)
  return value => run(null, value, () => {})
}



const makeUpperCaseAndShout = toFunc(
  toUpperCase,
  shout,
  console.log
)

makeUpperCaseAndShout("hello world")
```

Maybe we want to know when it is done executing, using a promise:

```js
const toPromise = (...operators) => {
  const run = pipe(...operators)
  return value => new Promise((resolve, reject) => 
    run(null, value, (err, value) => {
      if (err) reject(err)
      else resolve(value)
    })
   )
}

const makeUpperCaseAndShout = toPromise(
  toUpperCase,
  shout
)

makeUpperCaseAndShout("hello world").then(console.log)
```

Again, not extremly helpful, but hopefully you see the potential. Let us imagine:

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

### Map

```js
function map (operation) {
  return (err, value, next) => {
    if (err) next(err)
    else next(null, value)
  }
}
```

```ts
function map <Input, Output>(
  operation: (value: Input) => Output
): Operator<Input, Output> {
  ...
}
```

### CatchError

```js
function catchError (operation) {
  return (err, value, next) => {
    if (err) next(null, operation(err))
    else next(null, value)
  }
}
```

```ts
function catchError <Input, Output>(
  operation: (value: Error) => Output
): Operator<Input, Output> {
  ...
}
```


### Wait

```js
function wait (ms) {
  return (err, value, next) => {
    if (err) next(err)
    else setTimeout(() => next(null, value), ms)
  }
}
```

```ts
function wait <Input>(ms: number): Operator<Input, Input> {
  ...
}
```

### Filter
```js
function filter (operation) {
  return (err, value, next, final = next) => {
    if (err) next(err)
    else if (operation(value)) next(null, value)
    else final(null, value)
  }
}
```

```ts
function filter <Input>(
  operation: (value: Input) => boolean
): Operator<Input, Input> {
  ...
}
```

### Debounce

```js
function debounce (ms) {
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

```ts
function debounce <Input>(ms: number): Operator<Input, Input> {
  ...
}
```

### Pipe (async)

This operator transparently handles values that are promises. Meaning that for example **map** could return a promise or be an async function, and it would wait for it to resolve before moving on to next operator.

```js
function pipe (...operators) {
  return (err, value, next, final = next) => {
    if (err) next(err)
    else {
      let operatorIndex = 0
      const run = (runErr, runValue) =>
        operators[operatorIndex++](runErr, runValue, runNextOperator, final)

      const runNextOperator = (operatorError, operatorValue) => {
        if (operatorError) return next(operatorError)
        if (operatorIndex >= operators.length)
          return next(null, operatorValue)

        if (operatorValue instanceof Promise) {
          operatorValue
            .then((promiseValue) => run(null, promiseValue))
            .catch((promiseError) => next(promiseError, operatorValue))
        } else {
          try {
            run(null, operatorValue)
          } catch (operatorError) {
            next(operatorError, operatorValue)
          }
        }
      }

      runNextOperator(null, value)
    }
  }
}
```

```ts
function pipe<A, B>(
  aOperator: Operator<A, B>
): Operator<A, B>
function pipe<A, B, C>(
  aOperator: Operator<A, B>,
  bOperator: Operator<B, C>
): Operator<A, C>
function pipe<A, B, C, D>(
  aOperator: Operator<A, B>,
  bOperator: Operator<B, C>,
  cOperator: Operator<C, D>
): Operator<A, D>
// Continue this pattern to support as many operators you need
function pipe (...operators) {
  ...
}
```
