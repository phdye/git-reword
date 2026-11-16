# git-reword

Change the message of a commit that is already written.

    git reword <commit> -m "the message it should have had"
    git reword --edit HEAD~8
    git reword --edit 3f2a91c,8be0d47,HEAD~12,c41f7aa   # just these four
    git reword --list HEAD~8 -o todo.txt   # edit it, then
    git reword --from todo.txt

A message is part of the object that names a commit, so changing one mints a
new commit and every commit standing on it has to be minted again. That is
unavoidable. What is avoidable is anything else changing along the way, and
this tool's whole job is that nothing else does: a reworded commit carries the
same tree, the same author and author date, the same committer and committer
date, and the same parents as the one it replaces.

It replays rather than rebases. Each commit is read as a raw object, its
parents remapped, its message swapped, the result written back. Nothing is
re-applied and nothing is re-merged, so there is no conflict to resolve and no
hand-resolved merge to lose -- which `git rebase --rebase-merges` cannot
promise, because it makes each merge again.

## What it will not do

It will not reword a commit somebody else wrote, because the replay carries
the author across and the result would read as theirs while saying your words.
`--any-author` overrides it.

It will not reword a commit a remote-tracking branch already holds, because
publishing that rewrite takes a force push. `--published` rewrites locally
anyway. Nothing in this tool pushes.

It will not move a branch another worktree has checked out, and it will not
move a tag.

## What it does without being asked

It moves every local branch standing on a reworded commit, not only the one
checked out, and names them all in the plan first. Moving one and leaving the
others is how a repository comes to hold two spellings of the same history.

It copies each ref it moves under `refs/reword-backup/<stamp>/` before moving
it, and prints the `git update-ref` line that puts each one back.

It says what it is doing as it goes, one line per stage on stderr, with a
running count through the stages that take a git process per commit or per
ref. On a terminal the count is redrawn in place; in a log it is written every
few seconds. `-t` and `-q` silence it.

`-n` prints the plan and writes nothing.

## The table

`--edit` and `--list` write one entry per commit: a `commit` line at column
zero, the message indented four spaces beneath it. Change a message to reword
that commit; leave it alone, or delete the entry, and that commit is untouched.

    commit 7bf6489a5828  2026-03-14  A Person
        the subject as it stands

        and its body, indented the same way.

Comparison is exact, trailing whitespace included, so a commit nobody edited is
never reworded to the same text under a new id.

Given a range, the table offers every commit in it. Given commits separated by
commas, in any order, it offers those and nothing between them, oldest first,
which is the way to reword four commits scattered through fifty. All of them
are reworded in one replay.

Before the editor opens, `--edit` checks every commit in the table against
everything that would refuse the run (published, under a tag, in another
worktree's branch, another author), with the same progress counts, so a
refusal comes before any editing rather than after it. A published commit in
the table refuses even if you meant to leave it alone; narrow the selection,
or pass `--published`.

## Published commits

A commit a remote-tracking branch holds needs `--published`, and even then only
when every branch holding it can take a force push. Declare those per
repository, then fetch so the remote-tracking refs are current:

    git config --add reword.forcePushable 'origin/integration/*'
    git config --add reword.forcePushable 'origin/fix/*'
    git fetch --prune
    git reword --published --edit a1b2c3d,e4f5a6b

A commit on any branch not declared, `origin/main` or `origin/develop` say, is
refused with exit 8 before the editor opens, and no option gets past that.
Nothing is pushed; the force push afterwards is yours.

## Install

    ./install

which places it at `~/.local/bin/git-reword`, where git finds it as the
subcommand `git reword`. `./install --verify` reports drift; `./install
--prefix DIR` puts it elsewhere.

Called that way, ask for `git reword -h`, not `--help`: git turns `--help`
into a man page lookup for every subcommand, and there is no man page here.
`git-reword --help` works.

## Exit codes

0 ran, 1 failed, 2 usage, 3 not a repository or no such revision, 4 a ref is
checked out in another worktree, 5 another author's commit, 6 already
published, 7 a tag stands on it.

`git reword -h` has the rest.

## License

MIT. See `LICENSE`.
