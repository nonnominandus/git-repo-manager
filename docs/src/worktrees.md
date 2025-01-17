# Git Worktrees

## Why?

The default workflow when using git is having your repository in a single directory.
Then, you can check out a certain reference (usually a branch), which will update
the files in the directory to match the state of that reference. Most of the time,
this is exactly what you need and works perfectly. But especially when you're using
with branches a lot, you may notice that there is a lot of work required to make
everything run smootly.

Maybe you experienced the following: You're working on a feature branch. Then,
for some reason, you have to change branches (maybe to investigate some issue).
But you get the following:

```
error: Your local changes to the following files would be overwritten by checkout
```

Now you can create a temporary commit or stash your changes. In any case, you have
some mental overhead before you can work on something else. Especially with stashes,
you'll have to remember to do a `git stash pop` before resuming your work (I
cannot count the number of times where is "rediscovered" some code hidden in some
old stash I forgot about.

And even worse: If you're currently in the process of resolving merge conflicts or an
interactive rebase, there is just no way to "pause" this work to check out a
different branch.

Sometimes, it's crucial to have an unchanging state of your repository until some
long-running process finishes. I'm thinking of Ansible and Terraform runs. I'd
rather not change to a different branch while ansible or Terraform are running as
I have no idea how those tools would behave (and I'm not too eager to find out).

In any case, Git Worktrees are here for the rescue:

## What are git worktrees?

[Git Worktrees](https://git-scm.com/docs/git-worktree) allow you to have multiple
independent checkouts of your repository on different directories. You can have
multiple directories that correspond to different references in your repository.
Each worktree has it's independent working tree (duh) and index, so there is no
to run into conflicts. Changing to a different branch is just a `cd` away (if
the worktree is already set up).

## Worktrees in GRM

GRM exposes an opinionated way to use worktrees in your repositories. Opinionated,
because there is a single invariant that makes reasoning about your worktree
setup quite easy:

**The branch inside the worktree is always the same as the directory name of the worktree.**

In other words: If you're checking out branch `mybranch` into a new worktree, the
worktree directory will be named `mybranch`.

GRM can be used with both "normal" and worktree-enabled repositories. But note
that a single repository can be either the former or the latter. You'll have to
decide during the initial setup which way you want to go for that repository.

If you want to clone your repository in a worktree-enabled way, specify
`worktree_setup = true` for the repository in your `config.toml`:

```toml
[[trees.repos]]
name = "git-repo-manager"
worktree_setup = true
```

Now, when you run a `grm sync`, you'll notice that the directory of the repository
is empty! Well, not totally, there is a hidden directory called `.git-main-working-tree`.
This is where the repository actually "lives" (it's a bare checkout).

### Creating a new worktree

To actually work, you'll first have to create a new worktree checkout. All
worktree-related commands are available as subcommands of `grm worktree` (or
`grm wt` for short):

```
$ grm wt add mybranch
[✔] Worktree mybranch created
```

You'll see that there is now a directory called `mybranch` that contains a checkout
of your repository, using the branch `mybranch`

```bash
$ cd ./mybranch && git status
On branch mybranch
nothing to commit, working tree clean
```

You can work in this repository as usual. Make changes, commit them, revert them,
whatever you're up to :)

Just note that you *should* not change the branch inside the worktree
directory.  There is nothing preventing you from doing so, but you will notice
that you'll run into problems when trying to remove a worktree (more on that
later). It may also lead to confusing behaviour, as there can be no two
worktrees that have the same branch checked out. So if you decide to use the
worktree setup, go all in, let `grm` manage your branches and bury `git branch`
(and `git checkout -b`).

You will notice that there is no tracking branch set up for the new branch. You
can of course set up one manually after creating the worktree, but there is an
easier way, using the `--track` flag during creation. Let's create another
worktree. Go back to the root of the repository, and run:

```bash
$ grm wt add mybranch2 --track origin/mybranch2
[✔] Worktree mybranch2 created
```

You'll see that this branch is now tracking `mybranch` on the `origin` remote:

```bash
$ cd ./mybranch2 && git status
On branch mybranch

Your branch is up to date with 'origin/mybranch2'.
nothing to commit, working tree clean
```

The behaviour of `--track` differs depending on the existence of the remote branch:

* If the remote branch already exists, `grm` uses it as the base of the new
  local branch.
* If the remote branch does not exist (as in our example), `grm` will create a
  new remote tracking branch, using the default branch (either `main` or `master`)
  as the base

### Showing the status of your worktrees

There is a handy little command that will show your an overview over all worktrees
in a repository, including their status (i.e. changes files). Just run the following
in the root of your repository:

```
$ grm wt status
╭───────────┬────────┬──────────┬──────────────────╮
│ Worktree  ┆ Status ┆ Branch   ┆ Remote branch    │
╞═══════════╪════════╪══════════╪══════════════════╡
│ mybranch  ┆ ✔      ┆ mybranch ┆                  │
│ mybranch2 ┆ ✔      ┆ mybranch ┆ origin/mybranch2 │
╰───────────┴────────┴──────────┴──────────────────╯
```

The "Status" column would show any uncommitted changes (new / modified / deleted
files) and the "Remote branch" would show differences to the remote branch (e.g.
if there are new pushes to the remote branch that are not yet incorporated into
your local branch).


### Deleting worktrees

If you're done with your worktrees, use `grm wt delete` to delete them. Let's
start with `mybranch2`:

```
$ grm wt delete mybranch2
[✔] Worktree mybranch2 deleted
```

Easy. On to `mybranch`:

```
$ grm wt delete mybranch
[!] Changes in worktree: No remote tracking branch for branch mybranch found. Refusing to delete
```

Hmmm. `grm` tells you:

"Hey, there is no remote branch that you could have pushed
your changes to. I'd rather not delete work that you cannot recover."

Note that `grm` is very cautious here. As your repository will not be deleted,
you could still recover the commits via [`git-reflog`](https://git-scm.com/docs/git-reflog).
But better safe then sorry! Note that you'd get a similar error message if your
worktree had any uncommitted files, for the same reason. Now you can either
commit & push your changes, or your tell `grm` that you know what you're doing:

```
$ grm wt delete mybranch --force
[✔] Worktree mybranch deleted
```

If you just want to delete all worktrees that do not contain any changes, you
can also use the following:

```
$ grm wt clean
```

Note that this will not delete the default branch of the repository. It can of
course still be delete with `grm wt delete` if neccessary.

### Converting an existing repository

It is possible to convert an existing directory to a worktree setup, using `grm
wt convert`. This command has to be run in the root of the repository you want
to convert:

```
grm wt convert
[✔] Conversion successful
```

This command will refuse to run if you have any changes in your repository.
Commit them and try again!

Afterwards, the directory is empty, as there are no worktrees checked out yet.
Now you can use the usual commands to set up worktrees.

### Manual access

GRM isn't doing any magic, it's just git under the hood. If you need to have access
to the underlying git repository, you can always do this:

```
$ git --git-dir ./.git-main-working-tree [...]
```

This should never be required (whenever you have to do this, you can consider
this a bug in GRM and open an [issue](https://github.com/hakoerber/git-repo-manager/issues/new),
but it may help in a pinch.

