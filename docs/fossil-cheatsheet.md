# Fossil Cheat Sheet for Tcl/Tk

Fossil commands for everyday Tcl/Tk core work: one repository file, one checkout per maintained branch, autosync off, and every push done by hand.

## Setup

Clone from the mirror (fast), then point the remote at core (the canonical server, bandwidth-limited). `-A` sets the user name recorded on your check-ins. Here and below, `USER` is your login on core.tcl-lang.org.

```
fossil clone -A USER https://mirror1.tcl-lang.org/tk ~/tcltk/tk.fossil
fossil remote https://USER@core.tcl-lang.org/tk -R ~/tcltk/tk.fossil
fossil pull -R ~/tcltk/tk.fossil
```

One repo, many checkouts, one per branch:

```
mkdir ~/tcltk/tk && cd ~/tcltk/tk && fossil open ../tk.fossil main
mkdir ~/tcltk/tk9.0 && cd ~/tcltk/tk9.0 && fossil open ../tk.fossil core-9-0-branch
mkdir ~/tcltk/tk8.6 && cd ~/tcltk/tk8.6 && fossil open ../tk.fossil core-8-6-branch
```

Turn autosync off, so that commit, tag and ticket changes stay local until you push:

```
fossil settings autosync off       # per repo; check with: fossil settings autosync
fossil remote                      # show the remote URL
fossil user default USER           # if -A was forgotten
```

For Tcl, use the same commands with `tcl` in place of `tk`, naming the main checkout `~/tcltk/tcl9.1`.

## Branches and where you are

| Branch | Line | Checkout |
| --- | --- | --- |
| `main` | 9.1 development (formerly trunk) | `~/tcltk/tk`, `~/tcltk/tcl9.1` |
| `core-9-0-branch` | 9.0.x maintenance | `~/tcltk/tk9.0`, `~/tcltk/tcl9.0` |
| `core-8-6-branch` | 8.6.x maintenance | `~/tcltk/tk8.6`, `~/tcltk/tcl8.6` |

Release tags are the patchlevels: `core-9-0-2`, `core-8-6-17` and so on.

```
fossil branch current              # branch of this checkout
fossil status                      # checkout version, tags, changed files
fossil info [VERSION]              # details of a check-in: parent, children, tags
fossil branch ls [-a|-c|-r]        # open / all / closed / by recency
fossil leaves                      # open leaves (tips); -m: only branches with several
fossil timeline -n 20 -t ci        # last 20 check-ins
fossil timeline -b BRANCH -n 20    # one branch only
fossil timeline -p generic/tkFont.c   # history of one file
fossil finfo FILE                  # same, per-file view
fossil ui                          # web UI on the local repo
```

## Daily work

With autosync off, `pull` and `update` are separate steps: pull fetches into the repo, update moves the checkout.

```
fossil pull                        # fetch new artifacts (check-ins, tickets, tags)
fossil timeline -n 30              # see what arrived
fossil update                      # move checkout to the tip of its branch
fossil update -n                   # dry run: what would change
fossil update main                 # switch this checkout to another branch

fossil changes                     # edited / added / deleted files
fossil extras                      # untracked files (build leftovers, .orig)
fossil diff [FILE]                 # unified diff vs checkout version
fossil diff --branch NAME          # all changes made on branch NAME
fossil diff --from A --to B        # between two versions
fossil cat FILE -r VERSION         # file content at a version
fossil annotate FILE               # blame (also: fossil blame)

fossil add FILE / fossil rm FILE / fossil mv OLD NEW
fossil revert [FILE]               # discard local edits
fossil undo                        # undo the last update/merge/revert/stash pop
fossil stash save -m "wip"         # shelve edits; stash pop / ls / diff / drop
fossil clean -n                    # list what clean would delete (then think twice)
```

## Branching and committing

Start a fix branch from the tip of `main`, give it a descriptive name (`win-nonlatin-shortcuts`, not `bug-1234`), and push only once the commit is reviewed.

```
fossil pull && fossil update main  # start from current main
# ...edit, build, test...
fossil diff                        # review before committing
fossil commit -n --branch NAME -m "..."   # dry run
fossil commit --branch NAME -m "Fix [abcdef1234]: ..."   # first commit creates the branch
fossil commit -m "..."             # later commits stay on NAME

fossil branch new NAME BASIS       # alternative: create branch without committing
fossil amend HASH -m "new comment"  # fix a check-in comment (adds a tag, no rewrite)
fossil amend HASH --branch NAME    # move a check-in to another branch

fossil push                        # send everything unsent to core: irreversible
```

`[abcdef1234]` in a check-in comment links to the ticket or check-in with that hash. Fossil has no `unpublish`: once a check-in is pushed, the only fix is `fossil amend HASH --close` plus a follow-up commit. `--private` check-ins are never pushed (`--private` on push sends them anyway).

## Merging and backporting

Merge a fix branch into every line it fixes, in the checkout of that line. A plain `merge` brings in the branch's whole history, so a branch forked from `main` must be cherry-picked into 9.0 and 8.6.

```
# into main (branch forked from main)
cd ~/tcltk/tk && fossil update main
fossil merge -n NAME               # dry run, shows conflicts
fossil merge NAME                  # or: fossil merge --integrate NAME (closes NAME on commit)
fossil diff && make test
fossil commit -m "Fix [abcdef1234]: <ticket title>"

# backport into 9.0 / 8.6
cd ~/tcltk/tk9.0 && fossil update
fossil merge --cherrypick HASH     # one check-in; repeat for each
fossil commit -m "Fix [abcdef1234]: <ticket title>"   # same message as on main

# forward-merge a whole maintenance line (fix done on 8.6 first)
cd ~/tcltk/tk9.0 && fossil merge core-8-6-branch
fossil commit -m "Merge 8.6"

fossil merge --backout HASH        # reverse a check-in
fossil merge                       # no arg: merge a fork on this branch
fossil merge-base A B              # common ancestor (debugging merges)
```

A merge commit on `main`, `core-9-0-branch` or `core-8-6-branch` says what it fixes, like the check-ins of other core developers: "Fix [abcdef1234]: <ticket title>", not "Merge NAME". Only merges of a whole line say "Merge 8.6" (or "Merge 9.0").

After a 9.0 merge, add a line to `changes.md` (optional on 8.6). Then close the merged branches (`fossil branch close NAME`, or `fossil amend TIP --close`) and the tickets.

## Tags, CI and closing

In Tk, a tag starting with `core-` on a check-in makes the nightly GitHub mirror run CI on it. Tag only cross-platform changes, because each run costs CI minutes. Tcl supports the same tags, but there branches usually wait for a core developer instead.

```
fossil tag add core-NAME HASH      # local; then fossil push
# after midnight (Europe) sync: https://github.com/tcltk/tk/actions
fossil tag cancel core-NAME HASH   # once the run has started; push again

fossil tag list HASH               # tags on a check-in
fossil tag find core-NAME          # where a tag is
fossil branch close NAME           # close a merged branch
fossil branch reopen NAME
fossil branch ls -c                # closed branches
```

After each merge: close the tickets, close the merged branches, and cancel their `core-*` tags.

## Tickets from the command line

Ticket changes are artifacts like check-ins: they stay local until `fossil push` and can't be edited afterwards. Comment field is `icomment`, its markup field is `mimetype`.

```
fossil ticket show 0 "status='Open'"     # report 0 = all fields, with a filter
fossil ticket history UUID               # all changes of one ticket
fossil ticket list fields                # valid field names

fossil ticket set UUID login USER \
    mimetype text/x-fossil-wiki \
    icomment "Proposed fix in branch [/timeline?r=NAME|NAME]. Cause: ..."
fossil ticket set UUID status Closed resolution Fixed login USER \
    closer USER closedate $(fossil sql "SELECT julianday('now')")
fossil ticket add title "..." type Bug status Open \
    resolution None priority "5 Medium" assignee nobody closer nobody is_private 0 \
    login USER submitter USER \
    mimetype text/x-fossil-wiki icomment "..."
fossil test-wiki-render FILE             # preview wiki markup first
```

Common traps:

- The command line does not run the TH1 code of the web forms, so fields the forms fill in by themselves stay empty. When closing (status Closed or Deleted), set `closer` and `closedate` too, or the ticket has no close date. When adding a ticket, set the defaults of the new-ticket form (`resolution None`, `priority "5 Medium"`, `assignee nobody`, `closer nobody`, `is_private 0`).
- `cmimetype` is the mimetype of the ticket *description*. Setting it instead of `mimetype` posts the comment as unformatted plain text.
- Use `icomment`, not `+icomment`, when the same change sets other fields. Otherwise the artifact is malformed and silently never shows up.
- In wiki markup `[...]` is a link, even inside `<tt>` and `<code>`. Wrap Tcl substitutions like `[winfo id .]` in `<nowiki>` or `<verbatim>`.
- Use `[abcdef1234]` to link a check-in and `[/timeline?r=NAME|NAME]` to link a branch.
- Anonymous ticket edits on core.tcl-lang.org are moderated. They don't arrive by `pull` until approved, so check the web page.

## Git to Fossil

| Git | Fossil | Note |
| --- | --- | --- |
| `git fetch` | `fossil pull` | also brings tickets, tags, wiki |
| `git pull` | `fossil pull && fossil update` |  |
| `git switch B` | `fossil update B` | or a separate checkout per branch |
| `git switch -c B` | `fossil commit --branch B` | branch is born on first commit |
| `git add -A` | `fossil addremove` | no staging area; commit takes all changes |
| `git status` | `fossil changes`, `fossil extras` |  |
| `git log` | `fossil timeline` |  |
| `git show H` | `fossil info H`, `fossil diff --checkin H` |  |
| `git blame` | `fossil annotate` / `blame` |  |
| `git commit --amend` | `fossil amend H -m` | history is append-only |
| `git revert H` | `fossil merge --backout H` | then commit |
| `git cherry-pick H` | `fossil merge --cherrypick H` | then commit |
| `git branch -d` | `fossil branch close` | branches are never deleted |
| `git stash` | `fossil stash` |  |
| `git reset --hard` | `fossil revert` |  |
| `git merge-base` | `fossil merge-base` |  |

Useful links:

- [Git to Fossil Translation Guide](https://fossil-scm.org/home/doc/trunk/www/gitusers.md)
- [Fossil Quick Start](https://fossil-scm.org/home/doc/trunk/www/quickstart.wiki)
- [Getting Started With Working On The Tcl/Tk Core](https://wiki.tcl-lang.org/page/Getting+Started+With+Working+On+The+Tcl%2FTk+Core)
- `fossil help -a` lists every command, and `fossil help CMD` gives its options.
