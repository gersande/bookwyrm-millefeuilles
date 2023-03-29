# Updating millefeuilles

## Overview
This document describes the steps for updating millefeuilles when a new release of Bookwyrm is
available.

In brief, the process goes like this:
- Update the [bookwyrm-millefeuilles][repo] repo from the upstream bookwyrm project
- Back up the database
- Run the update script

## Update the repo
First, we will pull the new version into the millefeuilles repo, and resolve any conflicts.

**I recommend that you do this on your local machine.** Then, if something goes wrong, it won't
affect millefeuilles.cloud on the server; you can take your time to figure it out without
disrupting users.

### Prerequisite: have a copy of the repository on your local machine
The first time you do this, and any time you do this on a new computer, you will need to clone the
repo and configure the remotes. If you already did this, you can skip to the
[next step](#fetch-the-new-version).

#### Clone
In a terminal, in the directory where you want your copy of the repo to live, clone the repo:
```
git clone git@github.com:gersande/bookwyrm-millefeuilles.git
```

#### Check out the millefeuilles branch
Since we've been keeping our changes in the `millefeuilles` branch, check out that branch:
```
git checkout millefeuilles
```

#### Configure the remotes
At this point, your local copy of the repo has one *remote*, which is the `bookwyrm-millefeuilles`
repo on GitHub. By default, this remote is called `origin`. You can view the list of the remotes
with `git remote -v`. The output should look something like this:
```
$ git remote -v
origin	git@github.com:gersande/bookwyrm-millefeuilles.git (fetch)
origin	git@github.com:gersande/bookwyrm-millefeuilles.git (push)
```

In order to fetch updates from the upstream bookwyrm project, we need to add it as a remote, which
we will call `upstream`:
```
$ git remote add upstream git@github.com:bookwyrm-social/bookwyrm.git
```

### Fetch the new version
Usually, the new version you want to update to is a release tag on the Bookwyrm GitHub. Check the
[releases page][bookwyrm-releases] for the full list. Read the description of the release, in case
there are any special instructions for updating to this release.

For the examples in this document, I'll assume we are updating to vX.Y.Z.

#### Fetch the changes from upstream
First, we need to get vX.Y.Z from the upstream repo. We fetch all the latest changes:
```
$ git fetch upstream
```

This tells git to download all the latest changes from upstream, but it doesn't change our code
yet.

Next, we will merge version vX.Y.Z into our branch `millefeuilles`. Make sure you are on the branch
called `millefeuilles`. You can see what branch you are on by running `git status`. If the output
says that you are on some other branch, or if it doesn't say you're on a branch (for example, you
are in "detached head mode"), then checkout `millefeuilles`:
```
$ git checkout millefeuilles
```

Pull to make sure your local repo is up to date:
```
$ git pull
```

Merge vX.Y.Z into the current branch:
```
$ git merge --no-commit vX.Y.Z
```

Check the status:
```
$ git status
```

There are two things to look for here: conflicts, and changes to `docker-compose.yml`. If the
output of `git status` shows that `docker-compose.yml` has changed, we will need to analyze the
changes to figure out if we need to adapt them to our environment. This is true even there are no
conflicts in this file.

#### Resolve conflicts
If there were any conflicts, git probably told you when you ran the merge. At time of writing
(2023-03-28), we have only really changed the `docker-compose.yml` file, so there probably won't
be any conflicts.

If there are no conflicts, and no changes to `docker-compose.yml`, you can go to the
[next step](#commit-the-merge) now.

If there are any conflicts, they will be in the files at the bottom of the `git status` output,
underneath where it says "Unmerged paths:". In these files, the conflicting changes will be
between lines marked with '<<<<<<<' and '>>>>>>>'. In the example below, I'm merging v0.6.0, and
there is a conflict in the file `bookwyrm/settings.py`.
```python
env = Env()
env.read_env()
DOMAIN = env("DOMAIN")
<<<<<<< HEAD
VERSION = "0.5.5+millefeuilles"
=======
VERSION = "0.6.0"
>>>>>>> v0.6.0

RELEASE_API = env(
    "RELEASE_API",
    "https://api.github.com/repos/bookwyrm-social/bookwyrm/releases/latest",
)
```

Two versions of the conflicting section are present in the file. In this case, the conflicting
section is one line. The lines between `<<<<<<< HEAD` and `=======` are what this section looked
like in the current branch *before* merging the new version. The lines between `=======` and
`>>>>>> v0.6.0` are what this section looks like in the version from upstream.

This conflict arises because I changed this line right after updating to v0.5.5, and in upstream
it was also changed. So git is asking us which version we want to keep. In this case, we probably
want the line from upstream. We will therefore delete the HEAD section and keep the 'v0.6.0'
section. We also have to delete git's '<<<' and '>>>' markers, because those don't belong in the
code. They're just git's way of showing us where the conflict is. The result looks like this:
```python
env = Env()
env.read_env()
DOMAIN = env("DOMAIN")
VERSION = "0.6.0"

RELEASE_API = env(
    "RELEASE_API",
    "https://api.github.com/repos/bookwyrm-social/bookwyrm/releases/latest",
)
```

#### Commit the merge
Next, we will commit the merge. But first, if you resolved any conflicts, you need to mark those
files as resolved. You do this by adding them with `git add`. Continuing the example from above:
```
$ git add bookwyrm/settings.py
```

If you didn't have conflicts, you don't need `git add` anything.

Commit the merge:
```
$ git commit
```

Git will helpfully add a generic merge commit message. You can edit the message if you have
anything to note. 

#### Tag the commit
At this point, I recommend tagging this commit, so you can remember where you updated. If you
merged vX.Y.Z, call the tag vX.Y.Z+millefeuilles. (You can't call it vX.Y.Z, because that tag
already exists).
```
$ git tag vX.Y.Z-millefeuilles
```

#### Push the changes
Now you can push the changes to `bookwyrm-millefeuilles` on GitHub:
```
$ git push --tags origin millefeuilles
```
That will push both the new tag you created and the merge commit.

### Back up the database
*Note: looks like this has changed in v0.6.0, we might want to set up our own backups to a
DigitalOcean volume*

Connect to the millefeuilles.cloud server via SSH. Become the user `gersande` and cd into the
directory `/home/gersande/bookwyrm'.

Run the following command to take a database backup:
```
$ docker compose exec db bookwyrm-backup.sh
```

### Run the update script
Now run the update script:
```
$ ./bw-dev update
```


[repo]: https://github.com/gersande/bookwyrm-millefeuilles/tree/millefeuilles
[bookwyrm-releases]: https://github.com/bookwyrm-social/bookwyrm/releases
