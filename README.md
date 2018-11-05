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

## Example operators
