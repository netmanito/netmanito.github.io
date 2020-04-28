---
layout: post
title: Git 4 dummies
date: 2020-03-28
categories: linux tips
---

# Git 4 dummies

### BEFORE YOU START TO READ, PLEASE SEE THIS WEB !!

It's a very easy intro to GIT and it's in many languages

[http://rogerdudler.github.io/git-guide/](http://rogerdudler.github.io/git-guide/)

#### Create the Git master	

First create a user to hold Git repo, we'll call him git

`useradd git`

Then give your Git user a password:

`passwd git`

Ensure `/home/git` exists and create `.ssh` and .`ssh/authorized_keys` in it so well be able to ssh without password.

From your host, add your authorized key in the master

```
cat .ssh/id_rsa.pub | ssh git@123.45.56.78 "cat >> ~/.ssh/authorized_keys"
```

Now login as git user and create it’s git account and a new repository.

Don’t create the folder as it will be created by the command as an empty Git repository:

```
git@repohost:$ git init --bare test.git
```

You now have a new bare git repository, no working dir neither file tree will be stored on it. 
Now you have to clone the repository in your home directory and then you’ll be able to pull/push to git repo. 

### Working with Git	

#### Trying our new repo	

Every git user should first introduce himself to git, by running these two commands:

```
git@repohost:$ git config --global user.email "you@example.com"
git@repohost:$ git config --global user.name "Your Name"
```
Any client with ssh access to the machine can now clone the repository with

```
git@repohost:$ git clone git@repohost:/path/to/test.git
```

This has generated a directory called test where all data from git repo has been store.

Let’s try now adding a file

```
git@repohost:$ echo ""README from new repo" > README

git@repohost:$ ~/test $ git add README 

git@repohost:$ ~/test $ git commit -m "test"
[master (root-commit) 4404769] test
 1 file changed, 1 insertion(+)
 create mode 100644 README

git@repohost:$ ~/test $ git push origin master
Counting objects: 3, done.
Writing objects: 100% (3/3), 206 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@192.168.223.10:test.git
 * [new branch]      master -> master

git@repohost:$ ~/test $ git push origin master
Everything up-to-date
```

#### Clone and manage GIT from another host

Ok, we’ve created a repo and we’ve clone in the same host and same user, let’s try now from a remote machine.

```
[root@oracle_11gr2 ajsf]# git clone git@repohost:repo/test.git
[root@oracle_11gr2 ajsf]# echo "more data on README" >> README
[root@oracle_11gr2 ajsf]# git add README
[root@oracle_11gr2 ajsf]# git add * 
[root@oracle_11gr2 ajsf]# git commit -m "added howto ssh and some files"

[master 6b1a582] added howto ssh
 Committer: root <root@oracle_11gr2.local>
Your name and email address were configured automatically based
on your username and hostname. Please check that they are accurate.
You can suppress this message by setting them explicitly:

    git config --global user.name "Your Name"
    git config --global user.email you@example.com

If the identity used for this commit is wrong, you can fix it with:

    git commit --amend --author='Your Name <you@example.com>'

 1 files changed, 10 insertions(+), 0 deletions(-)
[root@oracle_11gr2 ajsf]#  git config --global user.name "editor"
[root@oracle_11gr2 ajsf]#  git config --global user.email "editor@whatever.com"
[root@oracle_11gr2 ajsf]# git commit -m "added howto ssh"
# On branch master
# Your branch is ahead of 'origin/master' by 1 commit.
#
nothing to commit (working directory clean)
```

Let’s upload the changes to repository

```
[root@oracle_11gr2 ajsf]# git push origin master
Counting objects: 5, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 439 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)
To git@repohost:repo/test.git
   e305104..6b1a582  master -> master
[root@oracle_11gr2 ajsf]#
```

#### Create a new Branch	

We going to create a new branch and upload it to the git repo. We first create the new branch at our local git directory and prepare all stuff.
Let’s assume we’re in our work computer and we have already cloned the repo on ~/test directory.

```
git@repohost:$ ~/test $  git status
# On branch master
nothing to commit, working directory clean
```

Create the new branch

```
git@repohost:$ ~/test $ git checkout -b preproduction
```

Include files you want to add, in our case ‘preproduction’ directory.

```
git@repohost:$ ~/test $ rsync -avz /etc/puppet/environments/preproduction .
git@repohost:$ ~/test $ git branch -a
  master
* preproduction
  remotes/origin/master
  remotes/origin/preproduction
```

Add files to git

```
git@repohost:$ ~/test $  git add *
git@repohost:$ ~/test $  git add preproduction/
git@repohost:$ ~/test $  git commit -m "preproduction modules added"
git@repohost:$ ~/test $  git push origin preproduction
```

#### Download only a certain Git branch

```
user@otherhost:dir/subidr$ git init
user@otherhost:dir/subidr$ git remote add preproduction git@repohost:repo/test.git
user@otherhost:dir/subidr$ git pull preproduction preproduction
```

#### Create a simple webserver to view and manage your git repository	

```
git@repohost:~/repo.git$ git instaweb --httpd=webrick --port=1234 --browser=firefox --start
git@repohost:~/repo.git$ git instaweb --stop
```

#### References

[https://training.github.com/kit/downloads/es/github-git-cheat-sheet.pdf](https://training.github.com/kit/downloads/es/github-git-cheat-sheet.pdf)

[http://rogerdudler.github.io/git-guide/](http://rogerdudler.github.io/git-guide/)

Git – the simple guide

[http://rogerdudler.github.io/git-guide/](http://rogerdudler.github.io/git-guide/)

Git – server basic Install

[https://help.ubuntu.com/lts/serverguide/git.html](https://help.ubuntu.com/lts/serverguide/git.html)


Git – Instaweb

[http://git-scm.com/docs/git-instaweb](http://git-scm.com/docs/git-instaweb)

[https://www.kernel.org/pub/software/scm/git/docs/git-instaweb.html](https://www.kernel.org/pub/software/scm/git/docs/git-instaweb.html)