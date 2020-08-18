---
title: Automatically Update your Codebase with GitHub Actions
description: Transpile Typescript into Javascript then commit the transpiled code back to the repository using GitHub Actions
date: 2020-08-06
tags:
  - GitHub
  - Actions
layout: layouts/post.njk
---
The [GitHub Marketplace](https://github.com/marketplace) holds a huge array of different tasks needed to complete just about anything to automate your workflow--and that's the problem. What **you** need to do falls between the cracks.

One thing GitHub got right when creating Actions is making it really easy to create and share your custom actions. However, one part that always niggled at me when creating actions was pushing the distribution folder with the compiled javascript to the repo. Normally transpiled code would be added to the `.gitignore` file rather than added to the codebase. It's easy for the non-transpiled and transpiled code to get out of sync.

Your GitHub Actions workflow file works by downloading the code in your repo and running in an environment defined by your `actions.yml` file. In the workflow file the keyword `uses` accepts an argument with a link to the repo using the format `<owner>/<repository>@<reference>` e.g. `- uses: peterjgrainger/action-changelog-reminder@master`. Until NodeJS is able to run typescript natively or someone builds a different engine that allows typescript as an input, we are stuck with transpilation. 

This blog post covers how I approached the problem of automatically pushing files to a git repository. This approach could be used for altering or adding any files not just this specific use case.


## Run Action on Each Push

The basic workflow file configures Actions to run on every push to the repository. This is required to make sure the transpiled code keeps up to date.

```yml
name: Push Transpiled Files
on: push
```

## Choose your runner

Your default runner for anything to do with Node should be `ubuntu-latest`. It's the easiest to work with when dealing with node.

```yml/5
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "run"
  run:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
```

### Clone Your Code

Next step in your workflow file is to clone your code locally to the runner. GitHub created the action `actions/checkout` so you know you can trust it.

```yml/7
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "run"
  run:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@main
```

## Setup Your Git Credentials

When you first use Git on a new computer there are a couple of settings that are required in the git config, your `name` and `email`. Your name and email are attached to the commit so you can look back through the history to see who broke the code.

The second bit of configuration is to authenticate. I normally do this through ssh but you can also configure git to use a token.

My first go at this was to run the Git commands directly.

