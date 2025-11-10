+++
date = '2025-07-20T12:09:55+08:00'
draft = false
title = 'Productivity Boost in Git'
+++

From time to time, I try to review if I have made use of the tooling at hand in an effective way.
Based on my experience with Git over the years, I want to put down some of my most frequently used utilities here.

### Making changes

I started using specialised `git switch` command for branch operations:
```shell
# switch to a new local branch
git switch -c <branch-name>
# switch to an existing local branch
git switch <branch-name>
# switch to the last branch I was at
git switch -
# switch to remote branch: 
# it auto-creates the local branch of the same name if not exist and tracks the given remote branch; all in one!
git switch <remote-branch-name>
```

The only time I still use `git checkout` is for file restoration:
```shell
# look at the file at specific version in the past
git checkout <commit-hash> -- <file-path>
```

Move file changes in and out of the next commit:
```shell
git add <file>
git reset <file> # keeps the local change
```

Quite often I happily made the commit and found out something I've missed to include.
```shell
# undo the last commit and retain all file changes
git reset --soft HEAD~1 # HEAD is the last commit in the current branch and HEAD~1 is the one before that
```

I tend to keep commit history clean and minimal. When I find small commits can be combined into one, I will do so.
```shell
# combine the last 3 commits into one commit
git reset --soft HEAD~3 && git commit -m "<message>"
```

Sometimes it is tempting to just `git reset --hard` to start all over again.
However, accident can happen, and it would be a lifesaver to also know how to restore from it.
```shell
git reset --hard ORIG_HEAD # it is the position of HEAD before the last git merge/rebase/reset
```

### Inspections

Look at branch information
```shell
# list down remote branches
git branch -r
# list local branches with their respective last commit (sometimes name alone doesn't tell)
git branch -v
```

Often I want to know which branch I am at right now
```shell
# but the command to do so requires a lot of typing
git branch --show-current
# I make a shorter alias for it
git config --global alias.curr 'branch --show-current'
# now here we go
git curr # master
```

To quickly narrow down the target when I look at the commit logs
```shell
# just show the last 3 commits
git log -3
# filter by commit message pattern
git log --grep="pattern"
# filter by commit author
git log --author="derek"

# minimal output that only shows commit hash and message
git log --oneline
# display visual branch lines just like Git history in IntelliJ
git log --graph
```

When I just want to scroll through the `git log` pages, press `b` to scroll up one page
and press whitespace to scroll down one page.

I will update this post if I encounter something else helpful in the future.
