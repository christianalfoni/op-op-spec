# op-op-spec
The op-op specification. A simple straight forward approach to functional programming

**Op-op** is just a signature that allows infinite async composition of logic.

**The signature**
```js
(err, value, next, final) => {}
```

**The type signature**
```ts
type Operator<Input, Output = Input> = (
  err: Error,
  value: Input,
  next: (err: Error, value: Output),
  final?: (err: Error, value: Output)
) => void
```

## Understand how it works

The simplest implementation of op-op is just to create a function:

```js
const doThis = (err, value, next) => {
  if (err) next(err)
  else next(null, value)
}
```

Which you could call like this:

```js
doThis(null, "foo", console.log) // null, "foo"
```

This does not make much sense right now, but this is the core building block of **op-op**. Let us use this signature to create a more powerful abstraction. Let us name it **pipe**:

```js
const pipe = (...initialOperators) => (err, value, next, final) => {
  if (err) next(err)
  else {
    const runNextOperator = (operators, err, val) {
      if (err) next(err)
      else if (operators.length) {
        const [nextOperator, ...remainingOperators]  = operators
        nextOperator(err, val, runNextOperator.bind(null, remainingOperators), final || next) 
      }
      else next(null, val)
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

## Example operators
