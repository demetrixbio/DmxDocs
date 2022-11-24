[[_TOC_]]

### Branch Naming

Gitlab creates branch names automatically, and they're pretty basic - just the issue number and name.  They'll trail out into one infinite list in git GUI apps, without regular cleanup.

We use something a little more complicated that plays nice with the GUI apps:

[two-digit year]/[two-digit month]/[three letter categorization]-description

The categorization is optional, and is drawn from the same list we use for commits.

Examples:

* `19/01/BUG-Node_dragging_fix`
* `19/01/ENH-Experiment_editor`
* `19/02/Inventory_Loader`
* `19/03/FEA-Material_Place_Time_API`

GUI git browsers will group these names into a tree per-month, which will give us some better granularity tracking active branches versus stale ones, and make pruning less of a necessity.

### Commit Naming

Commit convention for repo: first three letters of the type of commit in the subject

* `BCH` - started a new branch
* `BRK` - breaking change
* `BUG` - bug
* `CHK` - checkpoint (for a sequence of commits)
* `CNF` - changes to configuration
* `DAT` - update to data 
* `DOC` - doc change
* `DEP` - deprecation of code
* `ENH` - enhanced an existing feature
* `FEA` - added a new feature
* `MIN` - minor improvement
* `MNT` – maintenance (package updates, build script)
* `MRG` - commit after a merge
* `OOPS` - fixing mistake in a previous commit
* `REF` - refactoring some code
* `REL` - commit for an actual release
* `RMV` - removing a feature
* `SCH` - schema change (migrations for production, warehouse, etc)
* `TST` - changed tests
* `VER` - bump of version number
* `WIP` - work in progress
	
Example usage:

`BUG: Fixed off by one error in coordinate calculations`

makes it super easy to see thought process, you can see it in practice in the gsl public repo  https://github.com/AmyrisInc/GslCore/commits/master