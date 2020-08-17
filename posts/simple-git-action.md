---
title: Typescript to Javascript automatically in your Repo with GitHub Actions
description: Transpile Typescript into Javascript then commit the transpiled code back to the repository using GitHub Actions
date: 2020-08-06
tags:
  - GitHub
  - Actions
layout: layouts/post.njk
---
The [GitHub Marketplace](https://github.com/marketplace) holds a huge array of different tasks needed to complete just about anything to automate your workflow--and that's the problem. What **you** need to do falls between the cracks.

One thing GitHub got right when creating Actions is making it really easy to create a sharebut one part that always niggled at me is committing the transpiled code if you are using typescript. The part that makes them really easy to create and update over other CI systems is how they are referenced from the workflow. In the workflow file the keyword `uses` accepts an argument with a link to the repo using the format `<owner>/<repository>@<reference>` e.g. `- uses: peterjgrainger/action-changelog-reminder@master`. That repo must contain an action written in javascript to be interpreted correctly.

However, if you are created your action using typescript then you need to transpile your code before it can be used. There lies the issue. You will need to transpile your code and push to the repository before it can be used. Committing auto generated code is normally not a good idea. It's really easy to forget to do the traspilation and it's hard to tell for certain whether the typescript code is the same as the transpiled javascript.
