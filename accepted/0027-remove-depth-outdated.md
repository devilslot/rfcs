# Remove --depth from npm outdated

## Summary

Remove `--depth` in favor of a new option flag named `--all` that shows _all_ outdated packages in the tree.

## Motivation ("The Why")
Currently on npm v6 `npm outdated` has an optional depth flag to configure the maximum depth for checking outdated packages on the dependency tree. As it is right now, it takes an _integer_ value that represents the amount of nested levels deep it goes.

In practice, what most users care about is `--depth 9999` which should display a list of _all_ outdated dependencies on the tree. Also, npm v7 has already recently removed the `--depth` flag from `npm update` in favor of updating all nodes in the tree at every depth, which defeats the purpose of having the functionality to inspect outdated packages at specific depths if they are all going to be updated either way.

Removing `--depth` in favor of a new option flag named `--all` that will display all outdated dependencies on the tree makes for a clearer and more obvious output.

## Detailed Explanation
Going any level deep further than the top-level deps opens a complex scenario where a single dependency might need different versions to satisfy requirements from different packages depending on it. Basically, what this means is that the "wanted" version of a nested dependency could be either a  _single_ semver range or a _set_ of semver ranges. 

When `--depth` is used on v6, the same dependency and its "wanted" value will be displayed and repeated on console with their location as the sole way of differentiating its level of nesting. This current output can tend to clog the console since the command doesn't tell appart deduped dependencies and we end up with a lot of noise that could be summarized differently.

Ex.
```
➜  arborist git:(master) npm outdated --depth 9999 tar
Package  Current  Wanted  Latest  Location
tar       4.4.13  4.4.13   6.0.2  @npmcli/arborist > @npmcli/run-script > node-gyp
tar        6.0.1   6.0.2   6.0.2  @npmcli/arborist > pacote > cacache
tar        6.0.1   6.0.2   6.0.2  @npmcli/arborist > pacote > npm-registry-fetch > make-fetch-happen > cacache
tar        6.0.1   6.0.2   6.0.2  @npmcli/arborist > pacote

➜  arborist git:(master) npm ls tar
@npmcli/arborist@0.0.0-pre.11 /Users/claudiahdz/npm/arborist
├─┬ @npmcli/run-script@1.2.1
│ └─┬ node-gyp@6.1.0
│   └── tar@4.4.13
└─┬ pacote@11.1.0
  ├─┬ cacache@15.0.0
  │ └── tar@6.0.1  deduped
  └── tar@6.0.1
```

In this example, we only care about two versions of `tar`, `4.4.13` and `6.0.2`. Instead we are shown 4 lines of output.

Moreover, npm v6 currently has inconsistent behavior. Doing `npm outdated --depth 9999` displays all outdated packages that are direct children of the root package, missing out dependencies that are nested at other levels. This gives a wrong impression since we are not really displaying _all_ outdated dependencies of the tree. However, doing `npm outdated foo --depth 9999`, will indeed display all appearances of `foo` on the tree no matter if they are direct children of the root package or not. This is precisely the functionality that displaying all deps should have too (`npm outdated` with no flags). 

Finally, dropping `--depth` in favor of `--all` that will actually display _all_ outdated packages per dependency on the tree seems clearer and more useful to the end user than the original `--depth` flag.

Ex. `npm outdated --all`

For the tree:
```
root
+-- foo@1.0.0
|   +-- bar@1.0.0
+-- bar@2.0.0
```

With these dependencies for `root`:
```
{
  "foo": "^1.0.0",
  "bar": "^2.0.0"
}
```
and for `foo`:
```
{
  "bar": "^1.0.0",
}
```

Output:
```
Package                        Current   Wanted   Latest  Location
foo                              1.0.0    1.6.0    1.6.0  root
bar                              1.0.0    1.5.0    2.1.0  root > foo
bar                              2.0.0    2.1.0    2.1.0  root
```

## Implementation
Using Arborist, inspecting all nodes under a given root is straight-forward using `tree.inventory`. Looping through them and inspecting each node's `edgesIn` can easily give us access to said node dependants. This will simplify the current algorithm and avoid having to use recursion.

Finally, we could remove from the list duplicated dependencies that have the same wanted version to avoid clogging the terminal.

## Rationale and Alternatives
Since `npm update` will go ahead and update everything at any level, maybe there's no need to know in detail which nested dependencies are outdated and we could just show a message that certain top-level dependency has outdated deps.

We could also remove all-together the functionality if people are not paying much attention to outdated nested deps, Yarn only displays top-level outdated packages.

## Unresolved Questions and Bikeshedding
- What should we do with deduped packages?
- Are there better ways we could display packages "wanted" values?
