# Introduction

This shouldn't take long. Just briefly mention who I am and what I do.

# Installation

Users should already have Git installed, but it's a good idea to verify so that we can make sure that everyone is on the
same page.

## Verify Installation

- Open up your terminal.
- Enter `git --version`

## Create a GitHub account.

- Open https://github.com in your browser.
- Enter your Email address and click `Sign up for GitHub`
- [Wait for users to finish signing up then introduce them to the dashboard.]
- Next, you'll want to go to your profile page. Either:
    - Go to `https://github.com/{username}` or;
    - Click your avatar followed by `My Profile`.
- [Point users to the `Repositories` tab.]

## Why should we care?

- Brief description of how version control works.
    - Revision history
    - Branches
    - Merging
    - Conflicts

## Anatomy of a Git command.

- Git uses subcommands or verbs for its commands: `git {verb} {options}`
- You can use `git help` or `git -h` or simply `git` to get a list of common subcommands.
- You can use `git help -a` to get a list of all available subcommands.
- You can use `git help -g` to get a list of concept tutorials.

## Setting Up Git

- The command to configure git is `git config`.
- How would you know this?
    - It's a little tricky to find in the manual pages.
    - The Pro Git book (https://git-scm.com/book) will tell you.
    - If you try to commit a change to a Git repository without setting required configuration settings, git will remind
      you.
- Type `git config -h` to get a brief help message.
- Type `git config --help` to see a more verbose help message.
- Type `git config user.name "Your Name"` to tell it your name.
- Type `git config user.email youremail@example.org` to tell it your email address.
- Optional: Use `git config user.email ID+github-username@users.noreply.github.com` if you want to keep your email
  address private.
    - Caveat: this may not work with Git repositories that are not hosted on GitHub.
- Optional: tell git to normalize your line endings automatically
    - Use `git config --global core.autocrlf input` on Linux and Mac.
    - Use `git config --global core.autocrlf true` on Windows.
- Optional set a default editor
    - Use `git config --global core.editor nano`. (or `vim` or `emacs` or whatever).
- Type `git config --global init.defaultbranch main` to set the default branch name to `main`.
- Type `git config edit --global` to edit your configuration.
- Type `git config list --global` to list all of your configuration settings.

## Creating a Git Repository

- With respect to software, a repository is a place where you can store files.
- A Git repository is a location where you can store the entire revision history of a set of files.
- We're going to use the same directory we've been using for testing, so switch to that directory if you're not already
  there.
- Enter `git init` to create the repository.
- Everything in the directory tree underneath the repository directory is part of the repository.
- Enter `ls` to explore the directory. What gives? Nothing changed.
- Enter `ls -a` two check hiden files. Ah! There's a `.git` directory.
- The `.git` directory is where all of the information that Git maintains about a project is located.
- Enter `git status` to display the status of the repository or "repo."
- Looks like the data hasn't been added to the repository yet; let's add it.
- Enter `git add .` to add all files in the current working directory to the repo.
- Enter `git status` again to see what's going on; now the files have been staged, meaning they're ready to commit.
- Enter `git commit` to commit the files.
- The editor opens. This is where you get to annotate this revision of your files.
- The first line is a brief summary of the commit message.
- Subsequent lines can be used to provide more detail.
- Any line that begins with a hash mark is ignored.
- Save your commit message.
- [Describe the output of the command]
- Enter `git status` again; this time, there are no changes to commit.

## Tracking Changes

Now we get to create a file, but with the aid of version control.

### Create an Initial README File

- Type `nano README.md` in the shell.
- Type in a very brief description of the directory and save the changes.
- Type `ls` to verify that the file is present in the directory.
- Type `cat README.md` to verify its contents.
- Type `git status` to see the status of the repository.
- It indicates that the readme file hasn't been cheked in yet.
- Type `git add README.MD` to stage the file.
- Type `git status` again; the file has been staged.
- Type `git commit -m 'add the initial README file'`.
- [Briefly describe command output again.]
- Enter `git status` again; this time, there are no changes to commit again.
- Enter `git log`; notice that there are longer versions of the commit IDs here.

### Abandoning a Botched File Modification

Next, suppose we want to add descriptions of the data files in the repository.

- To make this easier, type `ls data > README.md` on the command line.
- Now type `cat README.MD` again. Uh oh! That's not good. The file was overwritten!
- All is not lost! Type `git restore README.md` to restore the file.
- Type `cat README.md` again to verify that the files were restored.
- Now let's try that command again `ls data >> README.md`.
- Type `cat README.md` again to see the contents of the file; much better!

### Adding Additional Changes

We're not quite done yet. Next, we need to finish our edits and commit them.

- Use `nano` or your favorite text editor to enter the descriptions of each of the files and save the changes.
- Typing `git diff` displays all of the differences.
- Now we can commit the changes! Enter `git commit -m "add data file descriptions"`
- We get an error saying that there's nothing to commit.
- Fortunately, the error message tells us what we have to do: run `git add` like we did before.
- But wait, what about this `git commit -a`.
- [Explore the command a little bit.]
- That will work fine in this case, so let's use it here.
- Type `git commit -am "add data file descriptions"`.

### Staging Area

We just noticed that Git will only commit data from the staging area. The stating area tells Git exactly which files you
want to place in a commit. This can be useful in several situations.

- There are some temporary files in the directory that you don't want to commit.
- You made multiple changes in different sets of files, and you want to create smaller commits so so that the commit
  messages will be more descriptive.
- You have some changes that aren't quite ready to be committed, but you want to temporarily preserve that version of
  the file before making additional related changes.

## Exploring History

This is one of the most useful features of version control systems.

### Comparing Current State to Previous Commits

- The most recent commit in the repository can always be referred to using the special identifier, `HEAD`.
- Let's make a change to the README to explore a bit.
- Edit the file by typing `nano README.md` and add a line.
- Next, type `git diff HEAD README.md`.
- Wait a minute! How is that different from just typing `git diff`?
    - Since that's the only file we cahnged, it's really not. `HEAD` is always the default commit for `git diff`.
    - The path was specified, so technically the equivalent command is `git diff README.md`
- What if we wanted to compare it to the previous commit?
- Type `git diff HEAD~1 README.md`.
- What if we wanted to compare it to the commit before that?
- Type `git diff HEAD~2 README.md`.

### Displaying Changes Made by a Commit

- You can use `git show` to display the changes made by a specific commit.
- For example, type `git show HEAD~2` on the command line.
- Similarly, you can type `git show HEAD~2 data/gapminder_all.csv` to list changes for just one file.

### Referring to Commits by ID

- Run `git log` to see a list of commits.
- Pick a commit and type `git show {commit-id}`
- Pick the first few characters of the commit ID and type `git show {first-fiew-chars}`
- This also works with `git diff`, for example: `git diff {first-few-chars} README.md`

### Reverting Unstaged Changes

- Run `git status`; it shows that the README has been modified.
- Let's revert the changes. Since they haven't been committed, we can use `git restore` to do that.
- Type `git restore README.md`
- type `git status` again; now it reports that there are no changes.

### Restoring From Other Commits

- Run `git restore -s {commit-id} {path}`: for example, `git restore -s HEAD~1 README.md`
- Run `git diff` to see the changes.
- Run `git status` to see the changes again.
- Maybe that wasn't such a great idea. Type `git restore README.md` again.

## Ignoring Things

What if we have some files that we don't want Git to track for us? Suppose we want to ignore all files with a `txt`
extension and everything in the scratch directory.

- Run `mkdir scratch`
- Run `touch a.txt b.txt c.txt scratch/a.csv scratch/b.csv scratch/c.csv`
- Run `git status`; the new folder and files show up as changes to be committed.
- We can tell Git to ignore these files by creating a `.gitignore` file.
- Run `nano .gitignore`
- Add `*.txt` to the file and save it.
- Run `git status` again; now only `.gitignore` and `scratch` show up.
- Run `nano .gitignore` again.
- Add `scratch/` and save it.
- Run `git status` again; now only `.gitignore` shows up.
- We should probably commit the `.gitignore` file.
- Run `git commit -am 'add .gitignore'`
- Wait; why'd we get the no-changes message?
- Ah, that's right! `-a` doesn't work for new files.
- Run `git add .gitignore`.
- Now run `git commit -m 'add .gitignore'`
- Now run `git status`
- What if we really, really want to ignore a file.
- Now run `git add a.txt`
- There's a warnig message saying that you can override the exclusion using the `-a` flag.
- You can also run `git status --ignored` to list filex that are excluded.

## Remote Repositories

This is where version control really starts to shine because it allows you to collaborate with others.

### Create a Remote Repository

- Log into GitHub
- In the upper right click the `Create New` dropdown then click `New Repository`.
- [Discuss options on the screen]
- Click `Create Repository`.

### Connect the Repositories

- Grab the SSH URI for the repository from GitHub.
- Run `git remote add origin {uri}`
- Run `git remote -v` to list the remotes.

### Create an SSH Key

- Run `ls -l ~/.ssh`
- If you have an SSH key, you can use it and skip creating a new one.
- If you don't hae one, you can create one now.
- Run `ssh keygen -t ed25519`
- It'll prompt you for a password; for security, it's a good idea to create a password and store it in a safe location.
- After entering and confirming the password, the new SSH key files will show up in your `~/.ssh` directory.

### Copy the SSH Key to GitHub

- Next run `cat ~/.ssh/id_ed25519.pub` and copy its contents to the clipboard.
- Back on GitHub click on your avatar followed by `Settings` then `SSH and GPG Keys`.
- Click `New SSH Key`.
- Give the new SSH key a name.
- Paste the contents of the public key file into the `Key` text box.
- Click `Add SSH Key`.
- Run `ssh -T git@github.com`

## Push Local Changes to the Remote

- Run `git push origin main`
- Refresh your browser on GitHub.
- What if you don't want to have to specify the remote and branch every time you push a change.
- You can set up a tracking branch for this by running `git push -u upstream main`.

## Collaborating

- Create a new clone of the same repository.
- Create a new file in the new clone.
- Commit and push the update.
- Switch back to the original clone.
- Pull the changes.

## Conflicts

- Modify the README in the new clone of the repo.
- Commit and push the changes.
- Modify the same section of the README in the original clone of the repo.
- Commit the change and try to push it; the push fails because of the conflict.
- Pull the changes.
- [Describe the conflict message.]
- Edit the file to resolve the conflict.
- Commit the changes.
- Push the changes.
- Go back to the new clone of the repo.
- Pull the changes.
