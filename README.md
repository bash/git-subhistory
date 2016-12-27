`git-subhistory` &mdash; Interchangeably merge in and split out subtree history

by Han <laughinghan@gmail.com>

### INTRO

`git-subhistory`, like `git-submodule` and `git-subtree`, manages subprojects
(call one "Sub") in a superproject (call it "Main") git repo. Like
`git-subtree` but unlike `git-submodule`, `git-subhistory` is stateless, you
pass "Sub's" subdirectory (call it `path/to/sub/`) as an argument to
`git-subhistory` but it's not specially marked in any way, "Sub's" files
are tracked directly by the "Main" repo like any other files, and no
other git tools know or care that `path/to/sub/` contains a subproject.

### WAIT BUT WHAT DOES THIS _DO_?

Well, you know how you can `git log path/to/sub/` to see the history
of just stuff in `path/to/sub/`? `git-subhistory split` (and
`git-subtree split`) is like that but creates a whole new commit history
of just that stuff, rooted inside `path/to/sub/`. This new commit
history of just "Sub" can be `git push`-ed to its own repo and developed
independently, or included in other projects.

Say that happens and commits are made on top of that new commit history.
Then `git-subhistory merge` merges those changes back into "Main's"
commit history by inverting `split`, creating new commits making those
same changes on top of the original "Main" commits, then merging them.
(Contrast `git-subtree merge`, which merges in such a way that the
resulting tree is good, but messes up the commit graph.)

### EXAMPLE

Let's say we have history like so:

```
  $ git log --graph --oneline --decorate --stat
  * c79da42 (HEAD, master) Add another Main thing
  |  another-Main-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * 6285d3c Add another Sub thing
  |  path/to/sub/another-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * 81ac521 Add a Sub thing
  |  path/to/sub/a-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * c242f3e Add a Main thing
     a-Main-thing | 1 +
     1 file changed, 1 insertion(+)
```

We can split out the commit history of just Sub in `path/to/sub/`:

```
  $ ../git-subhistory.sh split path/to/sub/ -b subproj
  Rewrite 6285d3c68b3eb5f0cd2cb8c376c0d83a0f2b7c0a (2/2)
  Ref 'refs/heads/subproj' was rewritten

  Split out history of path/to/sub/ to subproj (also SPLIT_HEAD)
  $ git log --graph --oneline --decorate --stat --all
  * c79da42 (HEAD, master) Add another Main thing
  |  another-Main-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * 6285d3c Add another Sub thing
  |  path/to/sub/another-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * 81ac521 Add a Sub thing
  |  path/to/sub/a-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * c242f3e Add a Main thing
     a-Main-thing | 1 +
     1 file changed, 1 insertion(+)
  * 13aa18a (subproj) Add another Sub thing
  |  another-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * 4970e20 Add a Sub thing
     a-Sub-thing | 1 +
     1 file changed, 1 insertion(+)
```

Notice 2 disconnected histories here, the untouched `master` branch, and
this new `subproj` branch, with the history of just the stuff inside
`path/to/sub/`. Say we push `subproj` to a public repo for just Sub, and
implausibly, our code isn't perfect and bugfixes are contributed to the
public repo. Now we've `git pull`-ed in upstream bugfixes:

```
  $ git log --graph --oneline --decorate --stat subproj
  * d535a7c (subproj) Fix Sub further
  |  fix-Sub-further | 1 +
  |  1 file changed, 1 insertion(+)
  * b7909ee Fix Sub somehow
  |  fix-Sub-somehow | 1 +
  |  1 file changed, 1 insertion(+)
  * 13aa18a Add another Sub thing
  |  another-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * 4970e20 Add a Sub thing
     a-Sub-thing | 1 +
     1 file changed, 1 insertion(+)
```

Here's where the magic happens: we can easily merge these changes into
Main **using the `Add another Sub thing` commit as the merge base**:

```
  $ ../git-subhistory.sh merge path/to/sub/ subproj
  Rewrite 6285d3c68b3eb5f0cd2cb8c376c0d83a0f2b7c0a (2/2)
  Ref 'SPLIT_HEAD' was rewritten

  Split out history of path/to/sub/ to SPLIT_HEAD

  Rewrite d535a7c1c90b42d66eb0938172538aa5fc878460 (2/2)
  Ref 'SUBHISTORY_MERGE_HEAD' was rewritten

  Merge made by the 'recursive' strategy.
   path/to/sub/fix-Sub-further | 1 +
   path/to/sub/fix-Sub-somehow | 1 +
   2 files changed, 2 insertions(+)
   create mode 100644 path/to/sub/fix-Sub-further
   create mode 100644 path/to/sub/fix-Sub-somehow
  $ git log --graph --oneline --decorate --stat
  *   2626aba (HEAD, master) Merge subhistory branch 'subproj' under path/to/sub/
  |\
  | * ff9f142 Fix Sub further
  | |  path/to/sub/fix-Sub-futher | 1 +
  | |  1 file changed, 1 insertion(+)
  | * e2d4c00 Fix Sub somehow
  | |  path/to/sub/fix-Sub-somehow | 1 +
  | |  1 file changed, 1 insertion(+)
  * | c79da42 Add another Main thing
  |/
  |    another-Main-thing | 1 +
  |    1 file changed, 1 insertion(+)
  * 6285d3c Add another Sub thing
  |  path/to/sub/another-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * 81ac521 Add a Sub thing
  |  path/to/sub/a-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * c242f3e Add a Main thing
     a-Main-thing | 1 +
     1 file changed, 1 insertion(+)
```

See that? The commits on `subproj` that fixed Sub after adding
`another-Sub-thing` were effortlessly transmuted into commits that fix
Sub inside `path/to/sub/` after adding `path/to/sub/another-Sub-thing`,
allowing them to merge cleanly like they clearly should, no commits
duplicated.

### DESCRIPTION

- `git-subhistory split <subproj-path> [(-b | -B) <subproj-branch>]`
  
  Literally just uses the `--subdirectory-filter` of `git-filter-branch`,
  which does pretty much the same thing as `git-subtree split`: it
  generates a completely new, synthetic commit graph of the history of
  just "Sub's" directory. Like the commit history shown by
  `git log --graph "path/to/sub/"`, it includes only commits that
  affected `path/to/sub/`, but each commit is rewritten so its root tree
  is the `path/to/sub/` subtree.

- `git-subhistory assimilate <subproj-path> <subproj-branch>`

  Invert `split`: look through the commits on `<subproj-branch>` for
  synthetic commits that were generated by splitting out some ancestor
  of `HEAD`, and then on top of the original "Main" commits those
  synthetic commits were generated from, generates more synthetic
  commits that make the same change as the "Sub" commits but to
  `path/to/sub/` in "Main" instead.  
  (Note: synthetic merge commits are tricky, because the "Main" trees of
   their parents may conflict. `git-subhistory` tries to create the
   simplest synthetic merge commits, but currently prioritizes
   guaranteeing that `ASSIMILATE_HEAD` be a clean merge into `HEAD`. This
   may change: mayhaps in the future simplicity of synthetic merge
   commits is prioritized, and `git-subhistory` merge will be made more
   sophisticated and will automatically resolve conflicts in the "Main"
   tree outside `path/to/sub/` in favor of `HEAD`.)  
  (Note: it is recommended to only split from/merge into the same Main
   branch. TODO: explain why merging in commits split from another
   branch can be like cherry-picking.)

- `git-subhistory merge <subproj-path> <subproj-branch>`
  
  Almost literally just
  `git-subhistory assimilate "$@" && git merge ASSIMILATE_HEAD`.

#### Notes

- working directory:  
  Unlike `git-submodule` and `git-subtree`, `git-subhistory` does NOT "need to
  [be run] from the toplevel of the working tree", run it wherever you
  damn well please, even inside `path/to/sub/` with `.` as `<subproj-path>`.

- `SPLIT_HEAD`:  
  All commands run `git-subhistory split` at some point, mutating
  `SPLIT_HEAD` to be `HEAD` split at `<subproj-path>`.

- grafts/replacement refs:  
  Synthetic commits are all created with `git-filter-branch`, which honors
  the `info/grafts` file and refs in `refs/replaces/`. Grafts and
  replacement refs will hence be permanently baked into the synthetic
  commit histories. Changing relevant ones will cause subsequent
  synthetic history to not match up at all.
  (Potential TODO: creating synthetic grafts and replacement refs for
   the synthetic history? But originals and synthetic grafts/replacement
   refs would have to be modified in sync.)

### COMPARISON WITH GIT-SUBTREE

I actually think `git-subtree` was halfway there, it got splitting pretty
much right, but then did merging wrong and had to needlessly complicate
splitting to deal with the broken merging. In fact, I started out trying
to fork `git-subtree`, intending to add an alternative way to merge,
until I discovered that `git-subtree split` was now more complicated
than `git filter-branch --subdirectory-filter`.

If you went through the example above with equivalent `git-subtree`
commands, the history after splitting would be identical (right down to
the hashes&mdash;the complication is only when you've `git-subtree merge`-ed
before into the branch being split), but after merging, you'd get:

```
  $ git log --graph --oneline --decorate --stat
  *   0584b9a (HEAD, master) Merge commit 'd535a7c1c90b42d66eb0938172538aa5fc878460'
  |\
  | * d535a7c (subproj) Fix Sub further
  | |  fix-Sub-further | 1 +
  | |  1 file changed, 1 insertion(+)
  | * b7909ee Fix Sub somehow
  | |  fix-Sub-somehow | 1 +
  | |  1 file changed, 1 insertion(+)
  | * 13aa18a Add another Sub thing
  | |  another-Sub-thing | 1 +
  | |  1 file changed, 1 insertion(+)
  | * 4970e20 Add a Sub thing
  |    a-Sub-thing | 1 +
  |    1 file changed, 1 insertion(+)
  * c79da42 Add another Main thing
  |  another-Main-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * 6285d3c Add another Sub thing
  |  path/to/sub/another-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * 81ac521 Add a Sub thing
  |  path/to/sub/a-Sub-thing | 1 +
  |  1 file changed, 1 insertion(+)
  * c242f3e Add a Main thing
     a-Main-thing | 1 +
     1 file changed, 1 insertion(+)
```

In this example, the commits adding `path/to/sub/a-Sub-thing` and
`path/to/sub/another-Sub-thing` to Main were split into commits adding
`a-Sub-thing` and `another-Sub-thing` to Sub just like you'd want.
`git-subhistory split` doesn't just serve the same purpose as
`git-subtree split`, it actually does almost exactly the same thing and
often generates identical commits with identical hashes, but is
implemented as a simple `filter-branch --subdirectory-filter`, whereas
`git-subtree` "manually" loops through history and calls `git commit-tree`.
I don't know why it does that, but conjecture that using `filter-branch`
it would be hard to deal with commits that are `merge -s subtree`-ed in
(see below), which are exactly where `git-subhistory`-generated commits
start diverging from `git-subtree`-generated commits.

You can see in the example that `git-subtree merge` does something
completely different from `git-subhistory merge`. `git-subtree merge`
actually does `merge --strategy subtree`, which is a fine and dandy
merge option that would let you include Sub in Main very similarly to
`git-subhistory`: in Main, you can `merge -s subtree` a commit from an
upstream Sub repo, and changes to Sub from there will be merged with
changes to `path/to/sub/` in Main. Importantly, as far as `git-merge` is
concerned it's just some strategy for merging the trees of two commits,
so it normally creates a merge commit whose parents are commits from
each of Main and Sub (notice in the example that the subproject files
`a-Sub-thing` and `another-Sub-thing` are at different paths between the
parents of `HEAD`). If merging in upstream changes is all you need, this
is awesome: next `merge -s subtree`, the last `merge -s subtree`-ed in
commit will be the merge base, and Sub's commit history shows up in
Main's commit history.

What is not awesome is that if you commit changes to Sub in the Main
repo, split them out into synthetic commits that get merged into an
upstream Sub repo, and then `merge -s subtree` from upstream Sub back
into Main, the synthetic commits are totally different from the Main
commits that changed Sub, neither is a fast-forward of the other, they
won't be a merge base, and they will both show up in Main's commit
history, essentially duplicating all commits to Sub made in Main (notice
in the example, 2 ancestors of `HEAD` create and add `a-Sub-thing` and
`another-Sub-thing`).

For this reason, most `git-subtree` tutorials I've seen actually recommend
using `--squash` when merging, which creates a synthetic commit combining
all the upstream changes to Sub since the last merge, which solves the
duplicate commit problem, but then, no upstream Sub commits are in
Main's history, Main commits are never a fast-forward of any upstream
Sub commits, and there will never be a merge-base. On projects I've
worked on where we wanted to both push and pull commits to and from an
upstream Sub repo (from and to a Main repo), we decided that was worse
than submodules and used submodules instead.
