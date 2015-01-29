# git submodule

Git submodules are implemented using two moving parts:

* the .gitmodules file
* and a special kind of tree object(with mode `160000`)

Git submodules put specific repo(what) to a specific path(where) in the super project
with the specific commit(when):

* what - `url` in .gitmodule files.
* where - `path` in .gitmodule files.
* when - a special __file__ object in the super projects, which has the commit SHA1 of 
  the submodules.

## How to track a branch of a submodule?

`-b <branch>` option in `git submodule add`, will add `branch` in .gitmodule file,
which affect the behavior of `git submodule update --remote`.
