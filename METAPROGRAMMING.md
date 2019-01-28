# Metaprogramming with decorators

Previously, metaprogramming within JavaScript was driven by use of dynamic operations that manipulate JavaScript objects, such as `Object.defineProperty`. ESnext upgrades classes with decorators, with the advantages:

* Users write class literals and explicitly annotate the piece of source code that is being manipulated, rather than spooky action-at-a-distance where a big object is passed into a special application-dependent class framework.
* Decorators can manipulate private fields and private methods, whereas operations on objects are unable to manipulate the definitions of private names.
* Decorators do their manipulations at the time the class is defined, rather than when instances are being created and used, so they may lead to patterns which are more amenable to efficient implementation.

Decorators can be used either to decorate a whole class or an individual class element (field or method).

## Basic Usage

Decorators are implemented as functions which take a JSON-like representation of class element(s) as an argument and return a possibly-modified form of that, optionally with additional class elements.

For example, suppose we want a simple decorator to change the placement of a property so that it will be created on the prototype rather than the instance:

```js
class Demo {
  @prototypeProp
  x = 1
}
```

The public field `x` above would be represented as the following class element descriptor, which will be passed into our decorator function:

```js
{
  kind: "field"
  key: "x",
  placement: "own",
  // property descriptor for Object.defineProperty
  configurable: false,
  enumerable: true,
  writable: true,
  initialize: () => 1
}
```

To implement our decorator function, we need to take the class element descriptor and change its `placement` property to `'prototype'`:

```js
function prototypeProp(elementDescriptor) {
  assert(elementDescriptor.kind == "field" || elementDescriptor.kind == "method");
  // Note: this decorator would generally be used for fields, not methods, because the
  // 'prototype' placement is already the default for non-static methods. So it would only
  // be useful for methods if you wanted to override the default placement or the placement
  // from a previous decorator.
  return {
    ...elementDescriptor,
    placement: 'prototype'
  }
}
```

Thanks to the `prototypeProp` decorator, `x` will now be placed on the prototype as if we had written the following code:

```js
class Demo {}
Demo.prototype.x = 1
```

## API

The signatures of the different kinds of decorator functions are as follows:

### Field Decorator

#### Parameters

A class element descriptor with the following properties:

```js
{
  kind: "field"
  key: String, Symbol or Private Name,
  placement: "static", "prototype" or "own",
  ...Property Descriptor (argument to Object.defineProperty),
  initialize: A method used to set the initial state of the field
}
```

For private fields or accessors, the `key` will be a Private Name--this is similar to a String or Symbol, except that it is invalid to use with property access `[]` or with operations such as `Object.defineProperty`. Instead, it can only be used with decorators.

#### Returns

A class element descriptor with the following properties (optional properties are indicated with `?`):

`{ kind, key, placement, ...descriptor, initialize, extras?, }`

The optional additional properties:

* `extras`: A property with additional class elements

### Method Decorator

#### Parameters

A class element descriptor with the following properties:

```js
{
  kind: "method"
  key: String, Symbol or Private Name,
  placement: "static", "prototype" or "own",
  ...Property Descriptor (argument to Object.defineProperty),
  method: The method itself
}
```

#### Returns

A class element descriptor with the following properties:

`{ kind, key, placement, ...descriptor, method, extras?, }`

### Class Decorator

#### Parameters

A class descriptor with the following properties:

```js
{
  kind: "class"
  elements: Array of all class elements
}
```

#### Returns

A class descriptor with the following properties:

```js
{
  kind: "class"
  elements: Possibly modified class elements (can include additional class elements)
}
```

### Hooks

The returned descriptor can be, or the `extras` array can contain, stand-alone initializers, of the following form:

```js
{
  kind: "hook"
  placement: "static", "prototype" or "own",
  start() { /* ... */ }
}
```

The start functions added here are just like field initializers, except that they don't result in defining a field.

These can be used, e.g., to make a decorator which defines fields through `[[Set]]` rather than `[[DefineOwnProperty]]`, or to store the field in some other place.

## Examples

The three decorators from [README.md](README.md) could be defined as follows:

```js
// Define the class as a custom element with the given tag name
function defineElement(tagName) {
  // In order for a decorator to take an argument, it takes that argument
  // in the outer function and returns a different function that's called
  // when actually decorating the class (manual currying).
  return function(classDescriptor) {
    let { kind, elements } = classDescriptor;
    assert(kind == "class");
    return {
      kind,
      elements,
      // This callback is called once the class is otherwise fully defined
      extras: [
        {
          kind: "hook",
          placement: "static",
          finish(klass) {
            window.customElements.define(tagName, klass);
          }
        }
      ]
    };
  };
}

// Create a bound version of the method as a field
//
// PLEASE NOTE: This is just an example implementation. There are open-source libraries
// with similar decorators, which are maintained unlike this example.
// See for example https://github.com/mbrowne/bound-decorator
function bound(elementDescriptor) {
  let { kind, key, method, enumerable, configurable, writable } = elementDescriptor;
  assert(kind == "method");
  let initialize =
    // check for private method
    typeof key == "object"
        ? function() { return method.bind(this) }
        // for public methods, defer lookup until construction to respect prototype chain
        : function() { return this[key].bind(this) };

  // Return both the original method and a bound function field that calls the method.
  // (That way the original method will still exist on the prototype, avoiding
  // confusing side-effects.)
  elementDescriptor.extras = [
    { kind: "field", key, placement: "own", enumerable, configurable, writable, initialize }
  ];
  return elementDescriptor;
}

// Whenever a read or write is done to a field, call the render()
// method afterwards. Implement this by replacing the field with
// a getter/setter pair.
function observed({kind, key, placement, enumerable, configurable, writable, initialize}) {
  assert(kind == "field");
  assert(placement == "own");
  // Create a new anonymous private name as a key for a class element
  let storage = PrivateName();
  let underlying = { kind, key: storage, placement, writable: true, initialize };
  return {
    kind: "method",
    key,
    placement,
    get() { return storage.get(this); },
    set(value) {
      storage.set(this, value);
      // Assume the @bound decorator was used on render
      window.requestAnimationFrame(this.render);
    },
    enumerable,
    configurable,
    extras: [underlying]
  };
}

// There is no built-in PrivateName constructor, but a new private name can
// be constructed by extracting it from a throwaway class
function PrivateName() {
  let name;
  function extract({key}) { name = key; }
  class Throwaway { @extract #_; }
  return name;
}
```

## Notes about the `@bound` decorator

For a rationale for this example decorator, see https://github.com/mbrowne/bound-decorator/blob/master/MOTIVATION.md
