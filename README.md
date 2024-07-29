![workflow](https://github.com/do-/node-subclassable-object-merger/actions/workflows/main.yml/badge.svg)
![Jest coverage](./badges/coverage-jest%20coverage.svg)

# Rationale

The public npm [registry](https://www.npmjs.com/) contains a plethora of modules implementing _deep_ (recursive) alternatives to the standard [Object.assign()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign).

Some of them are basically single functions with fixed logic hardly suitable for any extending, for instance:
* [deep-object-merger](https://github.com/DavideCometa/Deep-Object-Merger) (just a recursive `Object.assign()`)
* [multi-merge](https://github.com/eballci/multi-merge) (+something special about Boolean values)
* [object-merger](https://github.com/jarradseers/object-merger) (never mind Booleans, concatenate arrays)

Others offer to set up sophisticated custom rules using some languages other than plain ECMAScript:
* [decorated-merger](https://github.com/neckaros/decorated-merger) (TypeScript Decorations)
* [json-object-merge](https://github.com/sodaru/json-object-merge) (JSONPath)

The author of this module needed something very close to the basic naive implementation, but easily customizable by the means of standard ES. This is why `subclassable-object-merger` was created.

# Installation
```sh
npm install subclassable-object-merger
```

# Usage
```js
const {ObjectMerger} = require ('subclassable-object-merger')
const om = new ObjectMerger ()

const t1 = {
	name: 'users',
	label: 'System users',
	columns: {
		id: {TYPE: 'int'},
	},
	data: [
		{id: 1, label: 'admin'}
	],
	triggers: null,
}

const t2 = {
	name: 'users',
	label: undefined,
	columns: {
		id: {AUTO_INCREMENT: true},
		label: {TYPE: 'text'},
	},
	pk: 'id',
	data: [
		{id: 2, label: 'employee'}
	],
	triggers: {before_insert: 'RETURN;'},
}

om.merge (t1, t2) /* result:
{
	name: 'users',
	label: 'System users',
	pk: 'id',
	columns: {
		id:    {TYPE: 'int', AUTO_INCREMENT: true},
		label: {TYPE: 'text'},
	},
	data: [
		{id: 1, label: 'admin'},
		{id: 2, label: 'employee'}
	],
	triggers: {before_insert: 'RETURN;'},
})
*/
```

# Internals
## Instance properties
The only property is named `sum`: this is the object mapping type names (see `getType` below) to corresponding type specific adding functions. Simply put:
```js
this.sum = {
  array: (a, b) => a.concat (b),
  object: (a, b) => this.merge (a, b),			
  scalar: (a, b) => a // but complains unless a != b
}
```
## Constructor
Takes no parameters, just fills in `this.sum`. Inheriting classes may add some options, extend the set of types etc.

## Methods
### getType (a)
For a given non-null `a`, returns:
* `'array'` if `Array.isArray (a)`
* `'object'` if `typeof a === 'object'` (no advanced checker like [`is-plain-obj`](https://github.com/sindresorhus/is-plain-obj#is-plain-obj) nor [`is-plain-object`](https://github.com/jonschlinkert/is-plain-object) is in use here)
* `'scalar'` otherwise (e. g. for function valued `a`)

### add (a, b, k)
Does the main job here: calls `getType` for `a` and `b` and if the results are the same, merges `b` into `a`; otherwise, throws an error.

When `a` or `b` is `null` or `undefined`, returns the other argument (which may be `null` or `undefined` too).

The `k` parameter is a name of the outer objects properties whose values are `a` and `b`. It doesn't affect the result, but may appear in error messages. Descendant classes may use this argument for some special needs.

### merge (a, b)
This top level method copies `b`'s content into `a` by calling `add` for each of `b`'s keys. Both `a` and `b` must be plain Objects.

## Events
ObjectMerger inherits from [EventEmitter](https://nodejs.org/api/events.html#class-eventemitter). Though never using this feature on its own, it may be used as a message box by containing processes.