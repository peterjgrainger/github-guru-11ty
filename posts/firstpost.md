---
title: Testing GitHub Actions
description: Test your actions without a massive commit history
date: 2020-06-12
tags:
  - GitHub
  - Actions
  - Testing
layout: layouts/post.njk
---
Github actions are great. I think the concept of stringing loads of actions together rather than writing the same commands over and over in different pipelines. Finally, a pipeline without the frustration of committing code over and over to get it working.

Now we need to get good at creating actions 😃. The easiest way I've found to build actions without making a load of mistakes and with the least frustration is to make tests.

This post creates an action that adds a random monty python gif to to your pull request when you create a PR and the best way to write the test, because--why not!

## Action

