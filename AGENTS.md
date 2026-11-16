# git-reword

One command. It changes the message of a commit that is already written, and
every commit standing on that one, then moves the local refs that held them.
Nothing else about a commit changes: same tree, same author, same author date,
same committer, same committer date, same number of parents.

## Layout

    git-reword       the tool
    install          place it on PATH, and say what it replaced
    test/run         the checks, which are the tool's own --self-test
    spike/           the measurements the design rests on

## The one decision everything follows from

It replays; it does not rebase.

Each commit is read as a raw object, its parents remapped, its message swapped
where one was given, and the result written back with `hash-object`. Trees are
reused by id. Nothing is applied and nothing is merged, so there is no conflict
to resolve and no merge resolution to lose.

`git rebase --rebase-merges` looks like the obvious instrument and is the wrong
one. It re-creates a merge by performing the merge again, and a merge somebody
resolved by hand -- taking a hunk from neither side, or fixing the result
afterwards -- does not come back the same. The history then differs in content
from the one it replaced, which is exactly what a message-only change is not
supposed to do. `spike/replay-preserves-content` is the measurement.

`git filter-branch` has the right semantics and is deprecated and slow.
`git filter-repo` has them too, and drops the remotes and insists on a fresh
clone, which is a large side effect for changing three words.

## What it refuses, and why refusing is the feature

A commit somebody else wrote. Rewording puts your name on their words, and the
replay copies the author across, so the result reads as theirs while saying
what you wrote. `--any-author` if that is what is meant.

A commit a remote-tracking branch already holds. Publishing that rewrite takes
a force push, and that is the operator's decision, not a tool's. `--published`
rewrites locally and still pushes nothing; nothing in here pushes anything.

A branch another worktree has checked out. Moving the ref leaves that
worktree's index describing a commit its branch no longer names, and git then
reports the whole tree as changed.

A tag standing on a commit being reworded. A tag is a promise that an id will
not move. `--skip-tags` leaves it where it is, which keeps the pre-rewrite
commits reachable.

## Moving one ref is the bug worth guarding

Reword a commit that `main` and `develop` both carry, move only `main`, and the
repository now holds both spellings of the same history with nothing to say
which is meant. So every local ref standing on a reworded commit is found
first, named in the plan, and moved in one `update-ref --stdin` batch.

## Where a message can be edited in bulk

`--list` writes a table, `--edit` opens one in the editor, `--from` applies one
written earlier. An entry whose message comes back unchanged is left alone, not
rewritten to the same text under a new id.

That comparison is exact, which is why `render_table` does not trim trailing
whitespace. Azure DevOps writes a trailing space on every subject it generates,
and a table that trimmed one reported the commit as edited and reworded it for
nothing. There is a check for it in `--self-test`.

`--edit` and `--list` take a range or a comma-separated list of commits. A list
offers exactly the commits it names, in topological order, so commits far apart
need not drag everything between them into the table. A string with a comma
that resolves whole is taken as one revision, since a ref name may hold a
comma.

## Working on it

Run `./test/run` before committing. It builds a repository with a merge and a
side branch, rewords through it, and compares every tree, author and committer
by id.

The usage text and the argparse parser are two artifacts and nothing keeps them
in sync. Change one, change the other.

Python 3.6 is the floor: it runs under Cygwin's own interpreter, which is
3.6.9 in the rhel root. No walrus, no dataclasses, no `subprocess.run` keywords
added after 3.6.
