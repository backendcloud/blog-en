release time :2023-01-18 18:37

This article is not about the use of git, nor the source code of git, but reading this article will help you understand the underlying logic design of git. This article is about a folder, specifically the hidden folder `.git` of the local repository in the git distributed repository.

# The structure of the .git directory

    [root@iZ23nrc95u7Z ~]# mkdir git-dir
    [root@iZ23nrc95u7Z ~]# cd git-dir/
    [root@iZ23nrc95u7Z git-dir]# git init
    Initialized empty Git repository in /root/git-dir/.git/
    [root@iZ23nrc95u7Z git-dir]# ls -a
    .  ..  .git

*This hidden folder has the following folders and files:*

**hooks: is the directory that stores git hooks, which are scripts that are triggered when specific events occur. For example: before submitting, after submitting.**

    [root@iZ23nrc95u7Z git-dir]# cd .git/hooks/
    [root@iZ23nrc95u7Z hooks]# ls
    applypatch-msg.sample  commit-msg.sample  post-update.sample  pre-applypatch.sample  pre-commit.sample  prepare-commit-msg.sample  pre-push.sample  pre-rebase.sample  update.sample

**objects: It is an object library that stores various objects and contents of git.**

The file name is the SHA-1 hash checksum of the content of the workspace file managed by git, with 40 digits. The first two digits are used as the folder name, and the last 38 digits are used as the file name. The first 2 bits as folders are for quick indexing.

Because the checksum is based on the content of the file, the .git directory will not be saved repeatedly. For example, if a new branch is created based on a certain branch, duplicate files will not be saved repeatedly, and the modified file is also saved incremental information.

There are several types of information stored in the files in the directory objects, commit, tree (workspace folder information), blob (workspace file information).


**refs: is the directory that stores various git references, including branches, remote branches and tags.**

There are several directories under the refs directory: heads, remotes, tags

The heads directory stores the head information of the local branch. There are several branches in the local area corresponding to the files of several branch names. The content of the file is the commit id corresponding to the head of the branch.

remotes is the branch and head information of the remote repository.

The tags directory stores the tag and head information of the local repository. A tag corresponds to a file with the same name as the tag, and the content of the file is the commit id corresponding to the tag.

**config: is the configuration file at the code base level.**

The information in this file is exactly the same as that displayed by the git config –local -l command.

The information configured by the git config –local command will be added to this file. For example git config --local user.name "Your Name Here".

By modifying this file, the same effect as the git config –local command configuration can be achieved.

**HEAD: is the branch the codebase is currently pointing to.**

This file records the current branch information, eg ref: refs/heads/main.

If you use the git checkout command to switch branches, the information stored in the file will change accordingly.

If you modify this file, you can also achieve the same effect of switching branches as the git checkout command.

**description: This file is used by GitWeb.**

GitWeb is a CGI script (Common Gateway Interface, which is simply a program running on a web server, but triggered by browser input), allowing users to view git content on a web page. If we want to start GitWeb, we can use the following command:

git instaweb --start

Serve and open the browser http://127.0.0.1:1234, the page directly displays the current git repository name and description, the default description is as follows:

Unnamed repository; edit this file 'description' to name the repository.

The above output info is the content of the default description file. Editing this file will make the GitWeb description more friendly.


# The objects folder in the .git directory changes after common git command operations

**Add a file to the temporary storage area, changes in the objects folder**

    [root@iZ23nrc95u7Z git-dir]# ls
    [root@iZ23nrc95u7Z git-dir]# tree .git/objects/
    .git/objects/
    ├── info
    └── pack

    2 directories, 0 files
    [root@iZ23nrc95u7Z git-dir]# echo "this is file1" > file1.md
    [root@iZ23nrc95u7Z git-dir]# ls
    file1.md
    [root@iZ23nrc95u7Z git-dir]# cat file1.md
    this is file1
    [root@iZ23nrc95u7Z git-dir]# git add file1.md
    [root@iZ23nrc95u7Z git-dir]# tree .git/objects/
    .git/objects/
    ├── 43
    │   └── 3eb172726bc7b6d60e8d68efb0f0ef4e67a667
    ├── info
    └── pack

    3 directories, 1 file
    [root@iZ23nrc95u7Z git-dir]# git ls-files --stage
    100644 433eb172726bc7b6d60e8d68efb0f0ef4e67a667 0	file1.md
    [root@iZ23nrc95u7Z git-dir]# git cat-file -t 433eb172726bc7b6d60e8d68efb0f0ef4e67a667
    blob
    [root@iZ23nrc95u7Z git-dir]# git cat-file -p 433eb172726bc7b6d60e8d68efb0f0ef4e67a667
    this is file1





**Add a folder and create a new file in the folder, and save it to the temporary storage area, the changes in the objects folder**

    [root@iZ23nrc95u7Z git-dir]# ls
    file1.md
    [root@iZ23nrc95u7Z git-dir]# mkdir dir2
    [root@iZ23nrc95u7Z git-dir]# echo "this is file2" > dir2/file2.md
    [root@iZ23nrc95u7Z git-dir]# tree
    .
    ├── dir2
    │   └── file2.md
    └── file1.md

    1 directory, 2 files
    [root@iZ23nrc95u7Z git-dir]# git add dir2
    [root@iZ23nrc95u7Z git-dir]# tree .git/objects/
    .git/objects/
    ├── 43
    │   └── 3eb172726bc7b6d60e8d68efb0f0ef4e67a667
    ├── f1
    │   └── 38820097c8ef62a012205db0b1701df516f6d5
    ├── info
    └── pack

    4 directories, 2 files
    [root@iZ23nrc95u7Z git-dir]# git ls-files --stage
    100644 f138820097c8ef62a012205db0b1701df516f6d5 0	dir2/file2.md
    100644 433eb172726bc7b6d60e8d68efb0f0ef4e67a667 0	file1.md
    [root@iZ23nrc95u7Z git-dir]# git cat-file -t 433eb172726bc7b6d60e8d68efb0f0ef4e67a667
    blob
    [root@iZ23nrc95u7Z git-dir]# git cat-file -p 433eb172726bc7b6d60e8d68efb0f0ef4e67a667
    this is file1
    [root@iZ23nrc95u7Z git-dir]# git cat-file -t f138820097c8ef62a012205db0b1701df516f6d5
    blob
    [root@iZ23nrc95u7Z git-dir]# git cat-file -p f138820097c8ef62a012205db0b1701df516f6d5
    this is file2



**Submit content:**

git commit Submits the contents of the temporary storage area to the local warehouse.

    [root@iZ23nrc95u7Z git-dir]# git commit -m 'add file1.md and dir2/file2.md'
    [master (root-commit) b2d176a] add file1.md and dir2/file2.md
    2 files changed, 2 insertions(+)
    create mode 100644 dir2/file2.md
    create mode 100644 file1.md
    [root@iZ23nrc95u7Z git-dir]# tree .git/objects/
    .git/objects/
    ├── 01
    │   └── 54de200dfc71f16a7847f293d79b42b43995e6
    ├── 20
    │   └── d1d441485e8a194976cba515754890bffb32ad
    ├── 43
    │   └── 3eb172726bc7b6d60e8d68efb0f0ef4e67a667
    ├── b2
    │   └── d176a8c3f1d4f608b722cc3b79bdeeaebbdf06
    ├── f1
    │   └── 38820097c8ef62a012205db0b1701df516f6d5
    ├── info
    └── pack

    7 directories, 5 files

It is found that after the git commit command, in addition to the original two file blob objects, there are three more objects, namely the commit object, the commit as a whole tree object, and the tree object of the folder dir2 where file2.md is located.

    [root@iZ23nrc95u7Z git-dir]# git cat-file -t 0154de200dfc71f16a7847f293d79b42b43995e6
    tree
    [root@iZ23nrc95u7Z git-dir]# git cat-file -p 0154de200dfc71f16a7847f293d79b42b43995e6
    040000 tree 20d1d441485e8a194976cba515754890bffb32ad	dir2
    100644 blob 433eb172726bc7b6d60e8d68efb0f0ef4e67a667	file1.md
    [root@iZ23nrc95u7Z git-dir]# git cat-file -t 20d1d441485e8a194976cba515754890bffb32ad
    tree
    [root@iZ23nrc95u7Z git-dir]# git cat-file -p 20d1d441485e8a194976cba515754890bffb32ad
    100644 blob f138820097c8ef62a012205db0b1701df516f6d5	file2.md
    [root@iZ23nrc95u7Z git-dir]# git cat-file -t b2d176a8c3f1d4f608b722cc3b79bdeeaebbdf06
    commit
    [root@iZ23nrc95u7Z git-dir]# git cat-file -p b2d176a8c3f1d4f608b722cc3b79bdeeaebbdf06
    tree 0154de200dfc71f16a7847f293d79b42b43995e6
    author gitusername <gitusermail> 1674019979 +0800
    committer gitusername <gitusermail> 1674019979 +0800

    add file1.md and dir2/file2.md
    [root@iZ23nrc95u7Z git-dir]# git log
    commit b2d176a8c3f1d4f608b722cc3b79bdeeaebbdf06
    Author: gitusername <gitusermail>
    Date:   Wed Jan 18 13:32:59 2023 +0800

        add file1.md and dir2/file2.md
    [root@iZ23nrc95u7Z git-dir]# cat .git/COMMIT_EDITMSG
    add file1.md and dir2/file2.md




After commit, continue to modify the file content, or add new files and folders. Whether it is content modification or new creation, if the checksum changes, there will be more corresponding objects in the .git/objects directory, and if you commit again, there will be more commit objects and the entire tree objects of the commit.

So even if there are multiple versions, git will not save multiple copies of the same file content, but only the original file and incremental content. Each commit version has a clear structural snapshot, which can be restored to any commit. Create a new branch and commit in other branches. Through the branch information file and the HEAD information file of the branch, the commit history of different branches can be calculated.






