# DEMO

## Working locally with Git

Using the command line, you can easily integrate Git into your current workflow.

### Creating a local copy of the repository

Before we can work locally, we will need to create a clone of the repository.

When you clone a repository you are creating a copy of everything in that repository, including its history. This is one of the benefits of a DVCS like git - rather than being required to query a slow centralized server to review the commit history, queries are run locally and are lightning fast.

Let's go ahead and clone the class repository to your local desktop.

1. Navigate to the **Code** tab of the class repository on GitHub.
1. Click the green **Code** button.
1. Copy the **clone URL** to your clipboard.
1. Open your command-line application.
1. Retrieve a full copy of the repository from GitHub: `git clone <CLONE-URL>`
1. Once the clone is complete, navigate to the new directory created by the clone operation: `cd <REPOSITORY-NAME>`

### Our favorite Git command: `git status`

```shell-session
git status
On branch main
Your branch is up-to-date with 'origin/main'.
nothing to commit, working tree clean
```

`git status` is a command to verify the current state of your repository and the files it contains. Right now, we can see that we are on branch main, everything is up-to-date with origin/main and our working tree is clean.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyNDc3MDY5MTFdfQ==
-->