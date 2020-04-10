# Dependency resolution overrides

## Summary

Provide a mechanism for users to specify a different dependency resolution
at a specific point in the tree.

This supercedes and replaces [Implement a package override
option](../unaccepted/0009-package-overrides.md).

This feature was discussed in the Open RFC Call on 2020-04-08.

- [Notes](https://github.com/npm/rfcs/blob/latest/meetings/2020-04-08.md)
- [Recording](https://youtu.be/GTocZoiiu4k)

## Motivation

This addresses the following use cases:

- There is a bug that is yet to be fixed in a transitive dependency in our
  project's tree.  While awaiting the bugfix to be published, we wish to
  override any instances of the dependency with a known good version.
- There should only be a single copy of a given dependency in our tree, but
  npm's dependency resolution will result in a duplicate due to version
  conflicts.
- There is a transitive dependency X that exhibits a bug only when used in a
  particular way by another dependency Y.  We wish to replace that instance
  of X, but only when included in the dependency graph of Y.

## Detailed Explanation

Add a field to `package.json` called `overrides`, with the following shape:

```json
{
  "overrides": {
    "<selection package specifier>": "<result package specifier or overrides set>"
  }
}
```

The key is a package specifier with a name.  The value is either a package
specifier without a name, _or_ a nested overrides object.  Nested override
objects apply to dependency resolutions within the portion of the
dependency graph indicated by the key.

### No Cascading Overrides

The `overrides` key will only be considered when it is in the root
`package.json` file for a project.  `overrides` in installed dependencies
will _not_ be considered in dependency tree resolution.

`overrides` are not applied to overridden resolutions.  That is, if the
result of an override results in a package specifier that _would_ be
subject to an override at that level or higher, the second override is not
applied.  This prevents infinite regresses and loops, and greatly
simplifies the feature both from an implementation and user experience
point of view.

For example:

```json
{
  "overrides": {
    "foo@1.0.0": "2.0.0",
    "foo@2.0.0": "3.0.0",
    "foo@3.0.0": "1.0.0"
  }
}
```

In this case, a package that depends on `foo@1.0.0` will instead be given
`foo@2.0.0`.  A package that depends on `foo@2.0.0` will instead be given
`foo@3.0.0`.  What it will _not_ do is apply the `foo@1.0.0` override to
`foo@2.0.0`, and then consider whether any other overrides apply to it, and
so on.  (In this case, it would create an infinite regress, which is no
good.)

A more realistic and less contrived case where a cascade could be
desireable would be something like this:

```json
{
  "overrides": {
    "webpack@1.x": "2.x",
    "webpack@2.x": "2.7.0"
  }
}
```

In this case, we are saying that any `webpack@1` dependencies should be
bumped up to `webpack@2`, and furthermore, that any `webpack@2`
dependencies should be pinned to `webpack@2.7.0`.

In practice, since rules are applied once and not stacked or cascaded, any
webpack dependency that would resolve to a version matched by `2.x` will be
overridden to `2.7.0`.  But, any dependency that resolves to a version
matched by `1.x` will be set to whichever version happens to be the latest
`2.x` at the time of installation.

In order to produce the intended behavior described, the user would have to
either specify it twice:

```json
{
  "overrides": {
    "webpack@1.x": "2.7.0",
    "webpack@2.x": "2.7.0"
  }
}
```

Or make the dependency matching range wider:

```json
{
  "overrides": {
    "webpack@1.x || 2.x": "2.7.0"
  }
}
```

### Examples

To replace all versions of `x` with a version `1.2.3` throughout the tree:

```json
{
  "overrides": {
    "x": "1.2.3"
  }
}
```

If a bug is found in `x@1.2.3`, which is known to be fixed in `1.2.4`, then
bump _only_ that dependency (but anything that resolves to `1.2.2` or
`1.2.4` should be left alone.)

```json
{
  "overrides": {
    "x@1.2.3": "1.2.4"
  }
}
```

To replace all versions of `x@1.x` with version `1.2.3`, but only when used
as a dependency of `y@2.x`:

```json
{
  "overrides": {
    "y@2.x": {
      "x@1.x": "1.2.3"
    }
  }
}
```

To replace all instances of `underscore` with `lodash`:

```json
{
  "overrides": {
    "underscore": "npm:lodash"
  }
}
```

To force all versions of `react` to be `15.6.2`, _except_ when used by the
dependencies of `tap` (which depends on `ink` and requires `react@16`):

```json
{
  "overrides": {
    "react": "15.6.2",
    "tap": {
      "react": "16"
    }
  }
}
```

To use a known good git fork of `metameta`, but only when used as a
dependency of `foo` when `foo` is a dependency of `bar`:

```json
{
  "overrides": {
    "bar": {
      "foo": {
        "metameta": "git+ssh://git@gitserver.internal/metameta#known-good"
      }
    }
  }
}
```

## Prior Art and Alternatives

* [Yarn Selective Dependency
  Resolutions](https://classic.yarnpkg.com/en/docs/selective-version-resolutions/)

    This feature is very similar to yarn resolutions.  However, it avoids
    the following problems:

    - Using path/glob syntax for a graph traversal is somewhat challenging.
      In particular the `**` and `*` behavior is not well defined, and
      given the other areas where we use paths and globs (`files` and
      `workspaces` in particular) it sets an expectation that negating and
      advanced glob patterns would be supported.
    - Because yarn resolutions must indicate a specific _version_, it
      limits the use cases that we can support.

* [Prior Package Overrides RFC](../unaccepted/0009-package-overrides.md)

    Limiting a package name to only a specific version, and preventing it
    from being used in published projects, is unnecessarily limiting.

    Furthermore, the prior RFC was unclear whether dependencies with a
    `replace` field would have their replacements respected.  Allowing
    replacements in nested dependencies is hazardous, and a warning without
    an action is an antipattern, as it tends to train users out of
    expecting that warnings should be acted upon.

* [Pub
  `dependency_overrides`](https://www.dartlang.org/tools/pub/dependencies#dependency-overrides)

    Pub uses a dependency override option that allows defining a specific
    source or version.  As Dart does not use nested dependencies for
    conflict resolution, this is effectively the same as the feature this
    RFC describes, but without the nested override support.

* [Bower
  `resolutions`](https://github.com/bower/spec/blob/master/json.md#resolutions)

    Bower, like Pub, uses a flat dependency graph, and so conflicts must be
    resolved by the user.  When a user chooses an option in a dependency
    conflict, the resolution is saved to the `bower.json` file in the
    `resolutions` field, and used for future conflicts.  As Bower does not
    use nested dependencies for conflict resolution, this is effectively
    the same as the feature this RFC describes, but without the nested
    override support.

## Implementation

When building the ideal tree in `@npmcli/arborist`, take note of the
`overrides` defined in the root `package.json` file, and attach them to the
root node.

Three new fields are added to the `Node` object:

- `overrides` The overrides object applying to this node, or `null`.
- `overridesProvider` The node that supplied an overrides object to this
  one by its dependency relationship.
- `overridesRelevant` A boolean set initially to `false`, indicating that
  the overrides on this node were relevant in this branch of the dependency
  graph.

To flag a node's override as relevant:

- Set `node.overridesRelevant = true`.
- Flag `node.overridesProvider`'s overrides as relevant.

If any overrides exist on a given node, then:

- When resolving any given dependency, if it is not already subject to an
  override, test the resulting `resolved` value against the set of
  `override` keys.
- If there is a match, then:
    - If the value is a string, then replace the `Edge` object with a new
      edge representing the new spec, and re-resolve.  (Subsequent override
      matches will be ignored.)  Flag this node as having relevant
      overrides.
    - If the value is an object, then attach the value to the resulting
      dependency as `dep.overrides` and set `dep.overridesProvider` to the
      node with the dependency.
- If there is no match, then:
    - the `overrides` object is attached to the dep node.
    - the `overridesProvider` is set as a reference to the node with the
      dependency.
- When finding the placement of a node in the tree, deduplication can never
  be performed between two nodes with different `overrides` properties.
  That is, a node with an overrides object cannot be deduplicated against a
  node by the same name, and vice versa, unless they both are subject to
  the exact same overrides object.  (Either it must be nested, or in the
  case of peer dependencies, may be unresolvable.)  See `Cases causing
  excess duplication` below for explanation and example.
- When deciding which `edgesOut` to resolve (ie, the `problemEdges`
  method), we _must_ resolve any where there is an overrides entry with a
  package specifier matching the same package name, as it may be a conflict
  even if it would normally be deduplicated against a higher-level
  dependency in the tree.  It _may_ still end up being deduplicated against
  a version higher in the tree, in the current pass (if the overrides are
  identical), or later (if they are not relevant).

After the tree is built, address unnecessary duplication.

- For each node in the `root.inventory` set, where `node.overrides` is an
  object, and `node.overridesRelevant` is false, then set `node.overrides`
  to null, and add it to a list.
- For each node on the list, call `this.placeDep()` to attempt to place it
  in an ideal location, starting with its current location as a target.

### Cases causing excess duplication

Note that it is possible to _create_ dependency conflicts (and thus
excessive duplication) by using this feature, even though it is intended to
help _resolve_ dependency conflicts in an explicit manner.

Consider this dependency graph:

```
root -> (x, y, a, i)
x -> (a, i)
y -> (x, i)
a -> (b, c@1.x)
b -> (c@1.x, i)
c -> ()
i -> (j)
j -> ()
```

Normally resulting in this dependency resolution:

```
root
+-- a@1.0.0
+-- b@1.0.0
+-- c@1.3.0
+-- i@1.0.0
+-- j@1.0.0
+-- x@1.0.0
+-- y@1.0.0
```

Now, consider if the root has the following overrides:

```js
{
  "overrides": {  // <-- O
    "x": {        // <-- O'
      "a": {      // <-- O''
        "c": "1.2.3"
      }
    }
  }
}
```

There are 4 paths to `c` in the dependency graph:

```
root -> a -> b -> c
root -> a -> c
root -> x -> a -> b -> c
root -> x -> a -> c
root -> y -> x -> a -> b -> c
root -> y -> x -> a -> c
```

The last four paths should result in `c@1.2.3` from the override.  The
first two should result in `c@1.3.0`, as that is the latest available
version.

There should ideally _not_ ultimately be multiple copies of `i`, despite
the fact that it will be found via paths that are subject to different
`overrides` objects, because the override applied to `i` was never
relevant, as there is no case where the dependency graph includes `i -> x
-> a -> c`.

```
root -> i (subject to O)
root -> y -> i (subject to O)
root -> a -> b -> i (subject to O')
root -> x -> a -> b -> i (subject to O'')
root -> y -> x -> a -> b -> i (subject to O'')
```

The dep resolution algorithm will do the following:

- resolve root dependencies `(a, i, y, x)`, and add all to queue for
  dep resolution.  Place all 4 in the root `node_modules` folder.
    - apply `O` to `a`
    - apply `O` to `i`
    - apply `O` to `y`
    - apply `O'` to `x`, due to matching key
- resolve dependencies of a `(b, c)`
    - apply `O` to `b`.  No conflict, place in root `node_modules`.
    - apply `O` to `c`.  No conflict, place in root `node_modules`.
- Resolve dependencies of `a>b` (in root `node_modules`), `(c, i)`
    - `c` resolves to 1.3.0, receives overrides `O`.  No conflict, place in
      root `node_modules`
    - `i` receives overrides `O`.  Satisfying version with same `O`
      overrides object in root already, dedupe against existing dep.
- Resolve dependencies of `a>b>c` (in root `node_modules`), `()`
    - empty set, nothing to do
- Resolve dependencies of `i` (in `node_modules`), `(j)`
    - apply `O` to `j`.  No conflict place in root `node_modules`.
- Resolve dependencies of `i>j` (in `node_modules`), `()`
    - empty set, nothing to do
- Resolve dependencies of `x` (in `node_modules`), `(a, i)`
    - apply `O''` to `a`, due to matching overrides key.  Existing version
      in root `node_modules` has overrides of `O`, so place at
      `node_modules/x/node_modules/a`.
    - apply `O'` to `i`.  Existing version in root `node_modules` has
      overrides `O`, so place at `node_modules/x/node_modules/i`.
- Resolve dependencies of `y` (in `node_modules`), `(x, i)`
    - apply `O'` to `x`, due to matching key.  Duplicate version with same
      overrides object `O'` in root already, dedupe against existing dep.
    - apply `O` to `i`.  Duplicate version with same overrides object `O` in
      root already, dedupe against existing dep.
- Resolve dependencies of `x>a` (in `node_modules/x/node_modules`), `(b,c)`
    - apply `O''` to `b`.  Existing version in root `node_modules` has a
      different overrides object, so place at
      `node_modules/x/node_modules/b`.
    - `c` resolves to `1.3.0`, but override applies.  Flag `x>a` and `x` as
      having relevant overrides.  Different version of `c` in top level, so
      place at `node_modules/x/node_modules/c`.
- Resolve dependencies of `x>a>b` (in `node_modules/x/node_modules`), `(i, c)`
    - override `c` from `1.3.0` to `1.2.3` due to override match.  Same
      version present already in `node_modules/x/node_modules/c`, so dedupe
      against existing dep.  Flag `x>a>b` overrides as relevant.
    - Apply `O''` to `i`.  Existing version at
      `node_modules/x/node_modules/i` has overrides object `O'`, so place
      at `node_modules/x/node_modules/b/node_modules/i`.
- Resolve dependencies of `x>a>c`, `()`
    - empty set, nothing to do
- Resolve dependencies of `x>a>b>i` (j)
    - Apply `O''` to `j`.  Existing version at
      `node_modules/x/node_modules/j` has overrides object `O'`, so place
      at `node_modules/x/node_modules/b/node_modules/j`.
- Resolve dependencies of `x>a>b>i>j` ()
    - empty set, nothing to do

If we stopped and reified at this point, we'd have the following physical
tree:

```
root
+-- a
+-- b
+-- c (1.3.0)
+-- i (*)
+-- j (*)
+-- y
+-- x
    +-- a
    +-- b
    |   +-- i (*)
    |   +-- j (*)
    +-- c (1.2.3)
    +-- i (*)
    +-- j (*)
```

At no point was the `overridesRelevant` flag set for the nodes marked
`(*)`, so they are free to be deduped.

- For each dep in the tree, where `dep.overridesRelevant` is false, and
  `overrides` is an object
    - set `dep.overrides = null`
    - add it to a list
- Sort list in ascending order by `dep.depth`, then alphabetically by
  `dep.name`.
- For each item in the list, search the tree from the dep's parent for any
  packages by the same name which can be safely deduplicated against them
  (meaning: a compatible match for all edges resolving to the node, no
  incompatible shadowed dependencies, having no relevant `overrides` object
  attached), and remove those from the tree.

The resulting physical tree after deduplication would be:

```
root
+-- a
+-- b
+-- c (1.3.0)
+-- i (*)
+-- j (*)
+-- y
+-- x
    +-- a
    +-- b
    +-- c (1.2.3)
```

Only nodes `a`, `b`, and `c` under `x` are duplicated, because they are
subject to a relevant `overrides` setting.

## Questions and Bikeshedding

- The name "overrides" was chosen for the following reasons:

    - This feature is fundamentally different from Yarn `resolutions`, and
      closer in both effect and intent to `overrides` in Bower and Dep.  As
      there are fundamental semantic differences, it would not be possible
      to reliably translate a Yarn `resolutions` field to the format
      described here.  Therefor, it ought to be a different key.
    - Using this feature should be considered a hack in most cases,
      something that is done temporarily while waiting for a bug to be
      fixed, or to avoid excessive duplication issues caused by an overly
      strict meta-dependency specifier.  "Resolutions" sounds resolved and
      ok; "overrides" sounds like we're going against the package manager's
      recommendations.  (Not all npm users are native English speakers, but
      enough are that this connotation is worth considering.)

- Should `bundleDependencies` and `shrinkwrap` dependencies be subject to
  overrides?  It would be a change to those contracts, and impose
  additional implementation challenges, but my suspicion is that the user
  expectation is that they _would_ be overridden.

    If they are not, then it needs to be called out in the `overrides`
    documentation.

    If they are, then the "implementation" section of this RFC needs to be
    updated to consider the challenges involved.

- The assumption throughout this RFC (and in the deep dive call discussing
  it) is that nested dependency resolutions will be applied to _all_
  direct and transitive dependencies throughout the dependency graph from a
  given point.

    In other words, the override `{"x":{"c":"1.2.3"}}` will
    apply to any `c` that exists anywhere in the dependency graph starting
    from any `x`, without differentiating between `x -> a -> c` vs a direct
    dependendency `x -> c`.

    In "Yarn Selective Dependency Resolutions" terms, this is equivalent to
    an implicit `**` glob between every path portion in the key.

    While this makes the override more powerful, and simplifies the
    implementation, it also increases the risk that an override may apply
    to packages that the user did not intend it to.

    Supporting _both_ overrides for an entire branch of the package tree
    _and_ overrides limited to a direct dependency, would significantly
    increase the complexity of this feature.

    Using a nested object expression that does not support `**` or some
    equivalent, it would be extremely tedious and error-prone to expect the
    user to specify every path on the dependency graph where a module might
    be found.  Furthermore, it is arguably better in most cases to apply
    the override too broadly rather than too narrowly.
