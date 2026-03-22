# git cheat sheet

## setup & config

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor nvim
git config --global init.defaultBranch main
git config --list                        # show all config
git config --get remote.origin.url       # show remote URL
```

## init & clone

```bash
git init                                 # init repo in current dir
git init <dir>                           # init in new dir
git clone <url>                          # clone remote repo
git clone <url> <dir>                    # clone into specific dir
git clone --depth 1 <url>               # shallow clone (latest commit only)
```

## staging

```bash
git add <file>                           # stage a file
git add -A                               # stage all changes (new + modified + deleted)
git add -u                               # stage tracked changes only
git add -p                               # interactive hunk-by-hunk staging
git restore --staged <file>              # unstage a file (keep changes)
git restore <file>                       # discard working tree changes
git rm <file>                            # remove and stage deletion
git rm --cached <file>                   # stop tracking, keep on disk
git mv <old> <new>                       # rename and stage
```

## status & diff

```bash
git status                               # full status
git status -sb                           # short branch-aware status
git status --porcelain                   # machine-readable (good for scripts)
git diff                                 # unstaged changes
git diff --staged                        # staged changes (what will be committed)
git diff HEAD                            # all changes vs last commit
git diff <branch1>..<branch2>            # diff between branches
git diff --word-diff                     # word-level diff (good for prose)
git diff --stat                          # summary of changed files
```

## committing

```bash
git commit -m "message"                  # commit with inline message
git commit -v                            # commit with diff in editor
git commit --amend --no-edit             # amend last commit, keep message
git commit --amend                       # amend last commit, edit message
git commit --allow-empty -m "message"   # empty commit (useful for CI triggers)
```

## branches

```bash
git branch                               # list local branches
git branch -a                            # list all branches (local + remote)
git branch -v                            # list with last commit
git branch <name>                        # create branch (don't switch)
git branch -d <name>                     # delete merged branch
git branch -D <name>                     # force-delete branch
git branch -m <old> <new>               # rename branch
git switch <name>                        # switch branch (modern)
git switch -c <name>                     # create and switch (modern)
git checkout <name>                      # switch branch (classic)
git checkout -b <name>                   # create and switch (classic)
git checkout -b <name> origin/<name>    # track remote branch locally
```

## merging & rebasing

```bash
git merge <branch>                       # merge branch into current
git merge --no-ff <branch>              # force merge commit
git merge --squash <branch>             # squash into single staged change
git merge --abort                        # abort in-progress merge
git rebase <branch>                      # rebase current onto branch
git rebase -i HEAD~<n>                  # interactive rebase last n commits
git rebase -i --root                     # interactive rebase from first commit
git rebase --abort                       # abort rebase
git rebase --continue                    # continue after resolving conflict
git cherry-pick <hash>                   # apply a single commit
git cherry-pick <hash1>..<hash2>        # apply a range of commits
```

## remotes

```bash
git remote -v                            # list remotes
git remote add <name> <url>             # add remote
git remote remove <name>                 # remove remote
git remote rename <old> <new>           # rename remote
git remote set-url origin <url>         # change remote URL
git fetch --all --prune                  # fetch all remotes, prune stale refs
git pull                                 # fetch + merge
git pull --rebase                        # fetch + rebase (cleaner history)
git push                                 # push current branch
git push -u origin HEAD                  # push and set upstream (new branches)
git push --force-with-lease             # safe force push
git push origin --delete <branch>       # delete remote branch
git push --tags                          # push all tags
```

## log & history

```bash
git log                                  # full log
git log --oneline                        # compact one-line log
git log --oneline --graph --all --decorate  # visual branch graph
git log -n 20                            # last 20 commits
git log --author="name"                  # filter by author
git log --since="2 weeks ago"           # filter by date
git log --grep="pattern"                 # search commit messages
git log -S "string"                      # search for string added/removed (pickaxe)
git log -- <file>                        # log for a specific file
git log <branch1>..<branch2>            # commits in branch2 not in branch1
git show <hash>                          # show a commit's diff and metadata
git show <hash>:<file>                   # show a file at a specific commit
git blame <file>                         # show who changed each line
git blame -L 10,20 <file>               # blame a line range
```

## stash

```bash
git stash                                # stash working tree changes
git stash push -m "message"             # stash with a name
git stash list                           # list stashes
git stash pop                            # apply and drop top stash
git stash apply stash@{n}               # apply specific stash (keep it)
git stash drop stash@{n}                # delete a stash
git stash clear                          # delete all stashes
git stash branch <name>                  # create branch from stash
```

## undoing things

```bash
git restore <file>                       # discard working tree changes
git restore --staged <file>              # unstage (keep changes)
git reset HEAD~1                         # undo last commit, keep changes staged
git reset --soft HEAD~1                  # undo last commit, keep changes staged
git reset --mixed HEAD~1                 # undo last commit, unstage changes
git reset --hard HEAD~1                  # undo last commit, discard changes
git reset --hard HEAD                    # discard all working tree changes
git clean -fd                            # remove untracked files and dirs
git clean -fdx                           # also remove ignored files
git revert <hash>                        # create new commit that undoes a commit
git revert HEAD                          # revert the last commit
```

> **nuke alias:** `git reset --hard HEAD && git clean -fd` — back to clean HEAD in one shot.

## tags

```bash
git tag                                  # list tags
git tag <name>                           # create lightweight tag
git tag -a <name> -m "message"          # create annotated tag
git tag -a <name> <hash>                # tag a specific commit
git push origin <tag>                    # push a tag
git push --tags                          # push all tags
git tag -d <name>                        # delete local tag
git push origin --delete <name>         # delete remote tag
```

## submodules

```bash
git submodule add <url> <path>          # add submodule
git submodule update --init --recursive # init and fetch all submodules
git submodule update --remote           # update to latest remote
git clone --recurse-submodules <url>    # clone with submodules
```

## worktrees

```bash
git worktree add <path> <branch>        # check out branch in separate dir
git worktree list                        # list worktrees
git worktree remove <path>              # remove a worktree
```

## useful one-liners

```bash
# pull all subdirectory repos
find . -mindepth 1 -maxdepth 1 -type d -exec sh -c 'cd "{}"; git pull || echo "Failed in {}"' \;

# show files changed in last commit
git diff-tree --no-commit-id -r --name-only HEAD

# list all branches sorted by last commit
git branch --sort=-committerdate

# find which branch contains a commit
git branch --contains <hash>

# count commits by author
git shortlog -sn --all

# show size of repo objects
git count-objects -vH

# delete all local branches already merged to main
git branch --merged main | grep -v "^\* main" | xargs git branch -d

# interactively clean up last 5 commits
git rebase -i HEAD~5
```

## conflict resolution

```bash
git status                               # see conflicted files
# edit files, resolve <<<<< ===== >>>>> markers
git add <resolved-file>
git rebase --continue                    # or: git merge --continue
git mergetool                            # open configured merge tool
```

## .gitignore tips

```gitignore
*.log                   # ignore all .log files
/dist                   # ignore dist/ at repo root only
!important.log          # un-ignore a specific file
**/temp                 # ignore temp/ anywhere in tree
```

```bash
git check-ignore -v <file>              # explain why a file is ignored
git ls-files --others --ignored --exclude-standard  # list all ignored files
```
