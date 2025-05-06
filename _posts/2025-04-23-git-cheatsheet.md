---
title: "🧠 Git Cheatsheet"
date: 2025-04-23
description: "A comprehensive Git cheatsheet covering configuration, staging, commits, branching, merging, logs, tags, and more. Ideal for beginners and as a quick reference for experienced developers."
categories:
  - git
tags:
  - git
  - version control
  - command line
  - cheatsheet
  - developer tools
---

# Git Basics – A Practical Reference Guide

A personal cheat sheet and walkthrough for common Git commands and workflows, including setup, staging, committing, branching, merging, resetting, tagging, and more. Great for both beginners and as a quick reference for seasoned developers.

---

## 🧰 Git Basics

### Check Git Version
```bash
git --version
```

### Initialize a Git Repository
```bash
git init
```

### Clone a Repository
- Clone into current directory:
```bash
git clone https://github.com/reponame/repo.git .
```
- Clone into a new subdirectory:
```bash
git clone https://github.com/reponame/repo.git
```

### Git Configuration
- List global config:
```bash
git config --global --list
```
- Set username and email (can replace --global with --local to set values for specific repos):
```bash
git config --global user.name "Test User"
git config --global user.email test.user@gmail.com
```

### Add Changes to Staging
- Add all changes:
```bash
git add .
```
- Interactive patch staging (it will show you the chunks that have actually changed):
```bash
git add -p
```

### View Status
```bash
git status
```

---

## 🔍 Git Diff

### Compare Working Directory with Staging Area
```bash
git diff
```

### Compare Staging Area with Last Commit
```bash
git diff --cached
```

### Compare Working Directory with Last Commit
```bash
git diff HEAD
```

![Diff types](/assets/img/posts/2025-04-23-git-cheatsheet/diff-types.png)

### Compare Two Commits
```bash
git diff <commit>..<commit>
# Example
git diff dfgdf32..fg34s82
```

---

## 💾 Git Commit

### Basic Commit
```bash
git commit -m "This is a test commit"
```

### Add and Commit in One Step
```bash
git commit -am "Test commit in single command"
```

---

## ⏪ Git Reset & Restore

### Unstage All Files (Mixed Reset)
```bash
git reset
```

### Reset All (Hard Reset)
```bash
git reset --hard
```

### Unstage a Specific File
Remove a file from staging. So you did a `git add .` and now you want to remove a file from being commited.
```bash
git restore --staged nameoffile.txt
```

### Restore a File to Staged Version
Revert a single file in the `working directory` back from what is `staged`
```bash
git restore nameoffile.txt
```

### Restore from Last Commit (Working + Staging)
Revert a single file in `working` and `staged` from the last commit
```bash
git restore --staged --worktree nameoffile.txt
```

![Reset explained](/assets/img/posts/2025-04-23-git-cheatsheet/git-reset.png)

### Move HEAD Back
If you need to move your HEAD back to a previous commit you can use the below. This will not change the staging or working directory.

You can go back multiple commits using `~2`, `~3` etc

The `--soft` is telling it to only move the HEAD and not the staging and working directory
```bash
git reset HEAD~1 --soft
```

![Soft reset diagram](/assets/img/posts/2025-04-23-git-cheatsheet/git-reset-diagram.png)

### Reset to Specific Commit
Following on from above, you know the HASH of the commit you want to go back to, you can use the below
```bash
git reset <first x characters of commit> --soft

--soft = HEAD only
--mixed = HEAD and STAGE only
--hard = HEAD, Staging and Working directory (Everything)
```

![Reset types](/assets/img/posts/2025-04-23-git-cheatsheet/git-reset-specific.png)

---

## 🏷️ Git Tags

### Create Tag
```bash
git tag v1.0.0
```

### Tag a Previous Commit
```bash
git tag v0.9.1 <previous commit hash>
# Example #
git tag v0.9.1 fgwx62g
```

### List Tags
```bash
git tag --list
```

![Tags](/assets/img/posts/2025-04-23-git-cheatsheet/git-tag-1.png)

![Tags](/assets/img/posts/2025-04-23-git-cheatsheet/git-tag-2.png)

### Show Tag
```bash
git show v1.0.0
```

### Annotated Tag
```bash
git tag -a v0.0.1 <commit> -m "First version"
git show v0.0.1
```
![Tags](/assets/img/posts/2025-04-23-git-cheatsheet/git-tag-3.png)

**Tag Types Comparison**

- Lightweight:

![Lightweight](/assets/img/posts/2025-04-23-git-cheatsheet/git-tag-lightweight.png)

- Annotated:

![Annotated](/assets/img/posts/2025-04-23-git-cheatsheet/git-tag-annotated.png)

As you can see the annotated (first) is now of type `tag` and the lightweight tag is just of type `commit`

![Tag2](/assets/img/posts/2025-04-23-git-cheatsheet/git-tag-4.png)

### Push Tags to Remote
Just a note, tags are not pushed to the remote origin by default, you need to push tags specifically 
```bash
git push --tag
```

---

## 📜 Logs & History

### Full Commit Log
```bash
git log
```

### Show Patch for Commits
```bash
git log -p
```

### View Object Type
```bash
git cat-file -t <hash>
```

### View Object Content
```bash
git cat-file -p <hash>
```

### Graphical History View
```bash
git log --oneline --graph --decorate --all
```

---

## 🧭 Git Reflog

### Reference Logs
```bash
git reflog
```

---

## 🌐 Git Remote Origins

### Set Remote Origin
```bash
git remote add origin https://github.com/reponame/repo.git
```

### View Remote URLs
View a list of remote origins in a git folder
```bash
git remote -v
```

### Rename Current Branch
```bash
git branch -M main
```

### Push Main to Origin
```bash
git push -u origin main
```

### Pull from Remote
```bash
git pull
```
The above command is actually combining the below 2 commands together:
```bash
git fetch
git merge
```

---

## 🚫 Gitignore

Creating a file in the repo called .gitignore will tell git to ignore all files linked within.

Create a `.gitignore` file:
```gitignore
###################
# Compiled source #
###################
*.com
*.class
*.dll
*.exe
*.o
*.so

############
# Packages #
############
# it's better to unpack these files and commit the raw source
# git has its own built in compression methods
*.7z
*.dmg
*.gz
*.iso
*.jar
*.rar
*.tar
*.zip

######################
# Logs and databases #
######################
*.log
*.sql
*.sqlite

######################
# OS generated files #
######################
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

###############
# Directories #
###############
/logs/*
```

---

## 🌱 Git Branching & Merging

### Branch Commands

List all branches locally
```bash
git branch --list
```

View remote branches
```bash
git branch -r
```

View all branches (local and remote)
```bash
git branch -a
```

To create a new branch
```bash
git branch branchname
```

Switch to a new branch (switch is the preferred)
```bash
git checkout branchname
# OR #
git switch branchname
```
To create and checkout a branch in one step
```bash
git checkout -b branchname
```

To push a branch to a remote, -u sets up tracking between local and remote branch. Allows argumentless git pull in future.
```bash
git push -u origin branchname
```

### Merging Workflow
```bash
git switch main
git merge dev
```

### Delete Merged Branch
```bash
git branch -d dev
git push origin --delete dev
```

### No Fast-Forward Merge
```bash
git merge --no-ff branchname
```

### Rebase Workflow
```bash
git switch branch1
git rebase main
git rebase --continue
```

---

## 🔀 Pull Requests

- **Fork**: Copy a remote repo to your account.
- **Pull Request**: Ask original repo owner to pull your changes.

---

## ⚙️ Miscellaneous

### Create `gitgraph` Shortcut
```bash
function gitgraph {git log --oneline --graph --decorate --all}
```

### `.git` Folder Breakdown
- `objects/`: Stores content in compressed blobs
- `refs/`: Branches & tags
- `HEAD`: Points to current branch

---

> **Source**: [Git Internals Playlist](https://www.youtube.com/watch?v=hQJktcBzJUs&list=PLlVtbbG169nFr8RzQ4GIxUEznpNR53ERq&index=2)

