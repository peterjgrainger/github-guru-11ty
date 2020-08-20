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

## Transpile Your Code

The next step is to transpile your code. I've hidden the complexity of the transpilation by using an npm script `npm run build`. Below is the workflow with the added call to run the build script. The call to `npm run build` is highlighted.

```yml/8
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "run"
  run:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@main
    - run: npm run build
```

## Name and Email are Required Git Config Settings

Before you can commit any changes, `name` and `email` are required git config settings that need to be set. On commit, your `name` and `email` settings are added as metadata. By attaching this information you can later look back through the history--and see who broke your code. You can set these configuration parameters locally using the below on the command line

```bash
git config --global user.email "peter@grainger.xyz"
git config --global user.name "Peter Grainger"
```

The second bit of required configuration is authentication. On my laptop I would do this through ssh but another option is through a token. GitHub actions uses a token with a short lifetime and is accessible using the special reference `${{ secrets.GITHUB_TOKEN }}`. Git has a special command to set up this token. `git remote set-url origin https://peterjgrainger:${{ secrets.GITHUB_TOKEN }}@github.com/peterjgrainger/test-push-github.git`

## Run the Git Commands Directly in the Workflow

My first go at this was to run the above Git commands directly in the workflow. This approach works and you may choose to do it this way yourself.

```yml/9-15
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "run"
  run:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@main
    - run: npm run build
    - run: |
      git config --global user.email "peter@grainger.xyz"
      git config --global user.name "Peter Grainger"
      git commit -a -m "tested commiting via actions"
      git remote set-url origin https://peterjgrainger:${{ secrets.GITHUB_TOKEN }}@github.com/peterjgrainger/test-push-github.git
```

## Time to Tidy Up.

My son's favourite song at the minute is [Time to Tidy Up by Dave Moran](https://www.dailymotion.com/video/x6sg76r)--and that is exactly what we are going to do.

My workflow file looks a bit messy and manual. Time to tidy it up. My first stop was to have a look at the [GitHub Marketplace](https://github.com/marketplace) to see if there were any actions to handle the configuration side. Luckily there the [Configure Git Credentials](https://github.com/marketplace/actions/configure-git-credentials) action, with a very simple setup shown below.

```yml
- uses: oleksiyrudenko/gha-git-credentials@v1
      with:
        token: '${{ secrets.GITHUB_TOKEN }}'
```

## The Complete Workflow File

The following is the complete workflow. The steps are:
- Check out the code that triggered the push
- Configure Git with authorisation, name and email
- Transpile the Typescript into Javascript
- Stage and commit changes
- Push changes to GitHub 

```
name: Push Transpiled Files
on: push
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: oleksiyrudenko/gha-git-credentials@v1
      with:
        token: '${{ secrets.GITHUB_TOKEN }}'
    - run: |
      npm run build
      git commit -a -m "Auto Transpile"
      git push
```

## Workflow can be used for any Git Actions

You can copy this workflow to automatically perform any action that updates the repository. Want to automatically format your code or fix linting errors? No problem. Change the `npm run build` line to whatever command you need to update your code. Be careful though, some changes might require manual input and some linters could cause bugs if problems are automatically fixed.

