This doc describes core `jotai` bundle.

## atom

`atom` is a function to create an atom config. The atom config is an immutable object. The atom config itself doesn't hold an atom value. The atom value is stored in a Provider state.

```tsx
// primitive atom
function atom<Value>(initialValue: Value): PrimitiveAtom<Value>

// read-only atom
function atom<Value>(read: (get: Getter) => Value | Promise<Value>): Atom<Value>

// writable derived atom
function atom<Value, Update>(
  read: (get: Getter) => Value | Promise<Value>,
  write: (get: Getter, set: Setter, update: Update) => void | Promise<void>
): WritableAtom<Value, Update>

// write-only derived atom
function atom<Value, Update>(
  read: Value,
  write: (get: Getter, set: Setter, update: Update) => void | Promise<void>
): WritableAtom<Value, Update>
```

- `initialValue`: as the name says, it's an initial value which is the atom's going to return unless its value doesn't get changed.
- `read`: a function that's going to get called on every re-render. The signature of `read` is `(get) => Value | Promise<Value>`, and `get` is a function that takes an atom config and returns its value stored in Provider described below. Dependency is tracked, so if `get` is used for an atom at least once, the `read` will be reevaluated whenever the atom value is changed.
- `write`: a function mostly used for mutating atom's values, for a better description; it gets called whenever we call the second returned value of `useAtom`, the `useAtom()[1]`. The default value of this function in the primitive atom will change the value of that atom. The signature of `write` is `(get, set, update) => void | Promise<void>`. `get` is similar to the one described above, but it doesn’t track the dependency. `set` is a function that takes an atom config and a new value which then updates the atom value in Provider. `update` is an arbitrary value that we receive from the updating function returned by `useAtom` described below.

```tsx
const primitiveAtom = atom(initialValue)
const derivedAtomWithRead = atom(read)
const derivedAtomWithReadWrite = atom(read, write)
const derivedAtomWithWriteOnly = atom(null, write)
```

There are two kinds of atoms: a writable atom and a read-only atom. Primitive atoms are always writable. Derived atoms are writable if the `write` is specified. The `write` of primitive atoms is equivalent to the `setState` of `React.useState`.

### debugLabel

The created atom config can have an optional property `debugLabel`. The debug label will be used to display the atom in debugging. See [Debugging guide](../guides/debugging.md) for more information.

Note: Technically, the debug labels don’t have to be unique. However, it’s generally recommended to make them distinguishable.

### onMount

The created atom config can have an optional property `onMount`. `onMount` is a function which takes a function `setAtom` and returns `onUnmount` function optionally.

The `onMount` function will be invoked when the atom is first used in a provider, and `onUnmount` will be invoked when it’s not used. In some edge cases, an atom can be unmounted and then mounted immediately.

```tsx
const anAtom = atom(1)
anAtom.onMount = (setAtom) => {
  console.log('atom is mounted in provider')
  setAtom(c => c + 1) // increment count on mount
  return () => { ... } // return optional onUnmount function
}
```

Invoking `setAtom` function will invoke the atom’s `write`. Customizing `write` allows changing the behavior.

```tsx
const countAtom = atom(1)
const derivedAtom = atom(
  (get) => get(countAtom),
  (get, set, action) => {
    if (action.type === 'init') {
      set(countAtom, 10)
    } else if (action.type === 'inc') {
      set(countAtom, (c) => c + 1)
    }
  }
)
derivedAtom.onMount = (setAtom) => {
  setAtom({ type: 'init' })
}
```

## Provider

```ts
const Provider: React.FC<{
  initialValues?: Iterable<readonly [AnyAtom, unknown]>
  scope?: Scope
}>
```

Atom configs don't hold values. Atom values are stored in a Provider. A Provider can be used like React context provider. Usually, we place one Provider at the root of the app, however you could use multiple Providers, each storing different atom values for its component tree.

```jsx
const Root = () => (
  <Provider>
    <App />
  </Provider>
)
```

A Provider accepts an optional prop `initialValues` which you can specify
some initial atom values.
The use cases of this are testing and server side rendering.

```jsx
const TestRoot = () => (
  <Provider initialValues=[[atom1, 1], [atom2, 'b']]>
    <Component />
  </Provider>
)
```

A Provider accepts an optional prop `scope` which you can use for scoped atoms.
It works only for atoms with the same scope.
The recommendation for the scope value is a unique symbol.
The use case of scope is for library usage.

### Example

```jsx
const myScope = Symbol()

const anAtom = atom('')
anAtom.scope = myScope

const LibraryRoot = () => (
  <Provider scope={myScope}>
    <Component />
  </Provider>
)
```

## useAtom

```ts
// primitive or writable derived atom
function useAtom<Value, Update>(
  atom: WritableAtom<Value, Update>
): [Value, SetAtom<Update>]

// read-only atom
function useAtom<Value>(atom: Atom<Value>): [Value, never]
```

The useAtom hook is to read an atom value stored in the Provider. It returns the atom value and an updating function as a tuple, just like useState. It takes an atom config created with `atom()`. Initially, there is no value stored in the Provider. The first time the atom is used via `useAtom`, it will add an initial value in the Provider. If the atom is a derived atom, the read function is executed to compute an initial value. When an atom is no longer used, meaning all the components using it is unmounted, and the atom config no longer exists, the value is removed from the Provider.

```js
const [value, updateValue] = useAtom(anAtom)
```

The `updateValue` takes just one argument, which will be passed to the third argument of writeFunction of the atom. The behavior totally depends on how the writeFunction is implemented.
