# `git test`

`git-test` is a command-line script for running automated tests against commits in a Git repository. It is especially targeted at developers who like their tests to pass on *every* commit in a branch, not just the branch tip.

The best way to use `git test` is to keep a window open in a second linked worktree of your repository, and as often as you like run

    git test range master..mybranch

`git test` will test the commits in that range, reporting any failures. The pass/fail results of running tests are also recorded permanently in your repository as Git "notes" (see `git-notes(1)`).

If a commit in the range has already been tested, then by default `git test` reports the old results rather than testing it again. This means that you can run the above command over and over as you work, and `git test` won't repeat tests whose results it already knows. (Of course there are options to allow you to request explicitly that commits be retested.)

The test results are recorded by the *tree* that was tested, not the commit, so old test results remain valid even across some kinds of commit rewriting:

*   If commits are rewritten to change their log messages, authorship, dates, etc., the test results remain valid.
*   If consecutive commits are squashed, the results remain valid.
*   If a commit is split into two, only the first (partial) commit needs to be tested.
*   If some commits deep in a branch are reordered, the test results for commits built on top of the reordered commits often remain valid.

Of course this means that your tests should not depend on things besides the files in the tree. For example, whether your test passes/fails should *not* depend on the current branch name or commit message.


## Usage

### Defining tests

First define the test that you would like to run; for example,

    git test add "make -j8 && make test"

The string that you specify can be an arbitrary command; it is run with `sh -c`. Its exit code should be 0 if the test passes, or nonzero if it fails. The test definition is stored in your Git config.

### Test a range of commits

You can test a range of Git commits with a single command:

    git test range commit1..commit2

The test is run against each commit in the range, in order from old to new. If a commit fails the test, `git test` reports the error and stops with the broken commit checked out.

### Define multiple tests

You can define multiple tests in a single repository (e.g., cheap vs. expensive tests). Their results are kept separate. By default, the test called `default` is run, but you can specify a different test to add/run using the `--test=<name>`/`-t <name>` option:

    git test add "make test"
    git test range commit1..commit2
    git test add --test=build "make"
    git test range --test=build commit1..commit2

### Retrying tests and/or forgetting old test results

If you have flaky tests that occasionally fail for bogus reasons, you might want to re-run the test against a commit even though `git test` has already recorded a result for that commit. To do so, run `git test range` with the `--force`/`-f` or `--retest` options. If you want to forget old test results without retesting (e.g., if you change the test command), use `--forget`.

### Continue on test failures

Normally, `git test range` stops at the first broken commit that it finds. If you'd prefer for it to continue, use the `--keep-going`/`-k` option.

### For help

General help about `git test` can be obtained by running

    git test help

Help about a particular subcommand can be obtained via either

    git test help range

or

    git test range --help


## Best practice: use `git test` in a linked worktree

`git test` works really well together with `git worktree`. Keep a second worktree and use it for testing your current branch continuously as you work:

    git worktree add --detach ../test HEAD
    cd ../test
    git test range master..mybranch

The last command can be re-run any time; it only does significant work when something changes on your branch. Plus, with this setup you can continue to work in your main working tree while the tests run.

Because linked worktrees share branches and the git configuration with the main repository, test definitions and test results are visible across all worktrees. So you could even run multiple tests at the same time in multiple linked worktrees.


## Installation

Requirements:

* A recent Git command-line client
* A Python2 interpreter

Just put `bin/git-test` somewhere in your `$PATH`, adjusting its first line if necessary to invoke a Python2 interpreter properly in your environment.


## Ideas for future enhancements

Some other features that would be nice:

*   Be more consistent about restoring `HEAD`. `git test range` currently checks out the branch that you started on when it is finished, but only if all of the tests passed. We need some kind of `git test reset` command analogous to `git bisect reset`.

*   `git test run`, to run a test on a single commit.

*   Allow tests to be run against a dirty working tree (i.e., against uncommitted changes). Perhaps don't record the test results at all in this case. Perhaps, if all changes in the working tree are staged, record the test results against the tree SHA-1 of the staged changes.

*   `git test bisect`: run `git bisect run` against a range of commits, using a configured test as the command that `bisect` uses to decide whether a commit is good/bad.

*   `git test remove`: remove a test definition and its associated notes.

*   `git test prune`: delete notes for obsolete trees.

*   Continuous testing mode, where `git test` watches the repository for changes and re-runs itself automatically whenever the commits it is watching change.

*   Dependencies between tests; for example:

    *   Provide a way to say "if my `full` test passes, that implies that the `build` test would also pass".

    *   Provide a way to run the `build` test (and record the `build` test's results) as the first step of the `full` test.

*   Allow trees to be marked `skip`, if they shouldn't be tested (e.g., due to a known breakage). Perhaps allow the test script to emit a special return code to ask that the commit be marked `skip` (probably following the convention of `git bisect run`).

*   Support tests that depend on the *commit*, not the *tree*, that they are run against.

*   Make the script compatible with Python 3.


## License

`git test` is released under the GPLv2+ license. Pull requests are welcome at the project's GitHub page, https://github.com/mhagger/git-test


## Caveats and disclaimers

I've used this script for a while, but it surely still has bugs and rough edges. **Use `git test` at your own risk!**

For example, `git test` currently checks out commits willy-nilly, often leaving your repository in a "detached HEAD" state. This is not harmful per se, but it can be confusing, and if you are not careful it could potentially result in loss of work that is not on a branch.

To reduce the risk, **it is recommended that you run `git test` in a separate worktree**, which is more convenient anyway (see above for instructions). Note that the `git worktree` command was added in Git release 2.5, so make sure you are using that version of Git or (preferably) newer.

