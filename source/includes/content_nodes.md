# Content Nodes

After parsing, the `contents` value of each `YAML.Document` is the root of an [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) of nodes representing the document (or `null` for an empty document).

## Scalar Values

```js
class Node {
  comment: ?string,   // a comment on or immediately after this
  commentBefore: ?string, // a comment before this
  range: ?[number, number],
      // the [start, end] range of characters of the source parsed
      // into this node (undefined for pairs or if not parsed)
  spaceBefore: ?boolean,
      // a blank line before this node and its commentBefore
  tag: ?string,       // a fully qualified tag, if required
  toJSON(): any       // a plain JS representation of this node
}
```

For scalar values, the `tag` will not be set unless it was explicitly defined in the source document; this also applies for unsupported tags that have been resolved using a fallback tag (string, `Map`, or `Seq`).

```js
class Scalar extends Node {
  format: 'BIN' | 'HEX' | 'OCT' | 'TIME' | undefined,
      // By default (undefined), numbers use decimal notation.
      // The YAML 1.2 core schema only supports 'HEX' and 'OCT'.
  type:
    'BLOCK_FOLDED' | 'BLOCK_LITERAL' | 'PLAIN' |
    'QUOTE_DOUBLE' | 'QUOTE_SINGLE' | undefined,
  value: any
}
```

A parsed document's contents will have all of its non-object values wrapped in `Scalar` objects, which themselves may be in some hierarchy of `Map` and `Seq` collections. However, this is not a requirement for the document's stringification, which is rather tolerant regarding its input values, and will use [`YAML.createNode`](#yaml-createnode) when encountering an unwrapped value.

When stringifying, the node `type` will be taken into account by `!!str` and `!!binary` values, and ignored by other scalars. On the other hand, `!!int` and `!!float` stringifiers will take `format` into account.

## Collections

```js
class Pair extends Node {
  key: Node | any,    // key and value are always Node or null
  value: Node | any,  // when parsed, but can be set to anything
  type: 'PAIR'
}

class Map extends Node {
  items: Array<Pair>,
  type: 'FLOW_MAP' | 'MAP' | undefined
}

class Seq extends Node {
  items: Array<Node | any>,
  type: 'FLOW_SEQ' | 'SEQ' | undefined
}
```

Within all YAML documents, two forms of collections are supported: sequential `Seq` collections and key-value `Map` collections. The JavaScript representations of these collections both have an `items` array, which may (`Seq`) or must (`Map`) consist of `Pair` objects that contain a `key` and a `value` of any type, including `null`. The `items` array of a `Seq` object may contain values of any type.

When stringifying collections, by default block notation will be used. Flow notation will be selected if `type` is `FLOW_MAP` or `FLOW_SEQ`, the collection is within a surrounding flow collection, or if the collection is in an implicit key.

The `yaml-1.1` schema includes [additional collections](https://yaml.org/type/index.html) that are based on `Map` and `Seq`: `OMap` and `Pairs` are sequences of `Pair` objects (`OMap` requires unique keys & corresponds to the JS Map object), and `Set` is a map of keys with null values that corresponds to the JS Set object.

All of the collections provide the following accessor methods:

| Method                      | Returns   | Description                                                                                                                                                                                       |
| --------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| add(value)                  | `void`    | Adds a value to the collection. For `!!map` and `!!omap` the value must be a Pair instance or a `{ key, value }` object, which may not have a key that already exists in the map.                 |
| delete(key)                 | `boolean` | Removes a value from the collection. Returns `true` if the item was found and removed.                                                                                                            |
| get(key,&nbsp;[keepScalar]) | `any`     | Returns item at `key`, or `undefined` if not found. By default unwraps scalar values from their surrounding node; to disable set `keepScalar` to `true` (collections are always returned intact). |
| has(key)                    | `boolean` | Checks if the collection includes a value with the key `key`.                                                                                                                                     |
| set(key, value)             | `any`     | Sets a value in this collection. For `!!set`, `value` needs to be a boolean to add/remove the item from the set.                                                                                  |

```js
const map = YAML.createNode({ a: 1, b: [2, 3] })
map.add({ key: 'c', value: 4 })
  // => map.get('c') === 4 && map.has('c') === true
map.addIn(['b'], 5) // -> map.getIn(['b', 2]) === 5
map.delete('c') // true
map.deleteIn(['c', 'f']) // false
map.get('a') // 1
map.get(YAML.createNode('a'), true) // Scalar { value: 1 }
map.getIn(['b', 1]) // 3
map.has('c') // false
map.hasIn(['b', '0']) // true
map.set('c', null)
  // => map.get('c') === null && map.has('c') === true
map.setIn(['c', 'x'])
  // throws Error:
  // Expected YAML collection at c. Remaining path: x
```

For all of these methods, the keys may be nodes or their wrapped scalar values (i.e. `42` will match `Scalar { value: 42 }`) . Keys for `!!seq` should be positive integers, or their string representations. `add()` and `set()` do not automatically call `createNode()` to wrap the value.

Each of the methods also has a variant that requires an iterable as the first parameter, and allows fetching or modifying deeper collections: `addIn(path, value)`, `deleteIn(path)`, `getIn(path, keepScalar)`, `hasIn(path)`, `setIn(path, value)`. `getIn` and `hasIn` will return `undefined` or `false` (respectively) if any of the intermediate collections is not found or if the key path attempts to extend within a scalar value, but the others will throw an error in such cases. Note that for `addIn` the path argument points to the collection rather than the item.

## Alias Nodes

```js
class Alias extends Node {
  source: Scalar | Map | Seq,
  type: 'ALIAS'
}

const obj = YAML.parse('[ &x { X: 42 }, Y, *x ]')
  // => [ { X: 42 }, 'Y', { X: 42 } ]
obj[2].Z = 13
  // => [ { X: 42, Z: 13 }, 'Y', { X: 42, Z: 13 } ]
YAML.stringify(obj)
  // - &a1
  //   X: 42
  //   Z: 13
  // - Y
  // - *a1
```

`Alias` nodes provide a way to include a single node in multiple places in a document; the `source` of an alias node must be a preceding node in the document. Circular references are fully supported, and where possible the JS representation of alias nodes will be the actual source object.

When directly stringifying JS structures with `YAML.stringify()`, multiple references to the same object will result in including an autogenerated anchor at its first instance, and alias nodes to that anchor at later references. Directly calling `YAML.createNode()` will not create anchors or alias nodes, allowing for greater manual control.

```js
class Merge extends Pair {
  key: Scalar('<<'),      // defined by the type specification
  value: Seq<Alias(Map)>, // stringified as *A if length = 1
  type: 'MERGE_PAIR'
}
```

`Merge` nodes are not a core YAML 1.2 feature, but are defined as a [YAML 1.1 type](http://yaml.org/type/merge.html). They are only valid directly within a `Map#items` array and must contain one or more `Alias` nodes that themselves refer to `Map` nodes. When the surrounding map is resolved as a plain JS object, the key-value pairs of the aliased maps will be included in the object. Earlier `Alias` nodes override later ones, as do values set in the object directly.

To create and work with alias and merge nodes, you should use the [`YAML.Document#anchors`](#working-with-anchors) object.

## Creating Nodes

```js
const seq = YAML.createNode(['some', 'values', { balloons: 99 }])
// YAMLSeq {
//   items:
//    [ Scalar { value: 'some' },
//      Scalar { value: 'values' },
//      YAMLMap {
//        items:
//         [ Pair {
//             key: Scalar { value: 'balloons' },
//             value: Scalar { value: 99 } } ] } ] }

const doc = new YAML.Document()
doc.contents = seq
seq.items[0].comment = ' A commented item'
String(doc)
// - some # A commented item
// - values
// - balloons: 99
```

#### `YAML.createNode(value, wrapScalars?, tag?): Node`

`YAML.createNode` recursively turns objects into [collections](#collections). Generic objects as well as `Map` and its descendants become mappings, while arrays and other iterable objects result in sequences. If `wrapScalars` is undefined or `true`, it also wraps plain values in `Scalar` objects; if it is false and `value` is not an object, it will be returned directly.

To specify the collection type, set `tag` to its identifying string, e.g. `"!!omap"`. Note that this requires the corresponding tag to be available based on the default options. To use a specific document's schema, use the wrapped method `doc.schema.createNode(value, wrapScalars, tag)`.

The primary purpose of this function is to enable attaching comments or other metadata to a value, or to otherwise exert more fine-grained control over the stringified output. To that end, you'll need to assign its return value to the `contents` of a Document (or somewhere within said contents), as the document's schema is required for YAML string output.

<h4 style="clear:both"><code>new Map(), new Seq(), new Pair(key, value)</code></h4>

```js
import YAML from 'yaml'
import Pair from 'yaml/pair'
import Seq from 'yaml/seq'

const doc = new YAML.Document()
doc.contents = new Seq()
doc.contents.items = [
  'some values',
  42,
  { including: 'objects', 3: 'a string' }
]
doc.contents.items.push(new Pair(1, 'a number'))

doc.toString()
// - some values
// - 42
// - "3": a string
//   including: objects
// - 1: a number
```

To construct a `Seq` or `Map`, use [`YAML.createNode()`](#yaml-createnode) with array, object or iterable input, or create the collections directly by importing the classes from `yaml/seq` and `yaml/map`.

Once created, normal array operations may be used to modify the `items` array. New `Pair` objects may created by importing the class from `yaml/pair` and using its `new Pair(key, value)` constructor.

## Comments

```js
const doc = YAML.parseDocument(`
# This is YAML.
---
it has:
  - an array
  - of values
`)

doc.toJSON()
// { 'it has': [ 'an array', 'of values' ] }

doc.commentBefore
// ' This is YAML.'

const seq = doc.contents.items[0].value
seq.items[0].comment = ' item comment'
seq.comment = ' collection end comment'

doc.toString()
// # This is YAML.
//
// it has:
//   - an array # item comment
//   - of values
//   # collection end comment
```

A primary differentiator between this and other YAML libraries is the ability to programmatically handle comments, which according to [the spec](http://yaml.org/spec/1.2/spec.html#id2767100) "must not have any effect on the serialization tree or representation graph. In particular, comments are not associated with a particular node."

This library does allow comments to be handled programmatically, and does attach them to particular nodes (most often, the following node). Each `Scalar`, `Map`, `Seq` and the `Document` itself has `comment` and `commentBefore` members that may be set to a stringifiable value.

The string contents of comments are not processed by the library, except for merging adjacent comment lines together and prefixing each line with the `#` comment indicator. Document comments will be separated from the rest of the document by a blank line.

**Note**: Due to implementation details, the library's comment handling is not completely stable. In particular, when creating, writing, and then reading a YAML file, comments may sometimes be associated with a different node.

## Blank Lines

```js
const doc = YAML.parseDocument('[ one, two, three ]')

doc.contents.items[0].comment = ' item comment'
doc.contents.items[1].spaceBefore = true
doc.comment = ' document end comment'

doc.toString()
// [
//   one, # item comment
//
//   two,
//   three
// ]
//
// # document end comment
```

Similarly to comments, the YAML spec instructs non-content blank lines to be discarded. Instead of doing that, `yaml` provides a `spaceBefore` boolean property for each node. If true, the node (and its `commentBefore`, if any) will be separated from the preceding node by a blank line.

Note that scalar block values with "keep" chomping (i.e. with `+` in their header) consider any trailing empty lines to be a part of their content, so the `spaceBefore` setting of a node following such a value is ignored.
