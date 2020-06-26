---
title: Unit Test GitHub Actions
description: Create fast tests, that only test your logic.
date: 2020-06-12
tags:
  - GitHub
  - Actions
  - Testing
layout: layouts/post.njk
---
Github Actions are great. I think the concept of creating concise tasks then stringing them together is better than the duplication normally seen in pipelines.

To make good use of GitHub Actions pipeline, get good at making these building blocks. The path of learning, with the least frustration for me is through testing. The feedback cycle of pushing code, testing through GitHub, updating the action and pushing again is far too long. Testing Actions is often seen as a chore--and avoided. I'm going to show you how you can make testing fun and stress free! :)

This post describes the best way to organise your Jest test for a Typescript action. It doesn't cover how to test nor how the Actions work, it covers how to organise you tests to minimise frustration. Most of the actions I use involve interacting with GitHub in some way, so I'm going to cover my approach to testing these interactions. 

## Start with the Typescript Template

It's easiest to start with the typescript template. It has all the correct dependencies and testing all set up. Go to <https://github.com/actions/typescript-action> and press the `Use this template` button to create a copy of the repo locally. Clone that repo locally and we can begin creating the action and tests.

If you learn best by looking at code skip to the bottom for the full solution.

## Separate your Concerns

[Edsger W. Dijkstra](https://en.wikipedia.org/wiki/Edsger_W._Dijkstra) wrote a paper about Separation of concerns in, get this--1974. You can [read the paper online](http://www.cs.utexas.edu/users/EWD/transcriptions/EWD04xx/EWD447.html). It's full of programming gems, but the one I'm going to talk about here is SoC (Separation of Concerns). As described in this paper:

> The goal is to more effectively understand, design, and manage complex interdependent systems, so that functions can be reused, optimized independently of other functions, and insulated from the potential failure of other functions. - Edsger W. Dijkstra

In the context of GitHub Actions, this means separating the logic from the internals of how actions are called. By creating a self containing module we are making the code easier to test. Modular design is portable to use the same code in a webhook, CLI or running locally.

## You Don't Need to Test Everything

I have two modes: don't test at all--or test **everything**! Remind yourself, like a mantra, you **do not need to test everything** 

![test all the things](https://www.ghyston.com/media/1312/waatt.png)

Split your code into two parts. One part you test the other you don't. The part you test contains all your logic and the the part you don't test includes all the parts that are hard to test, like imports. Have as little code as possible in the non-tested part.

## Update Your main.ts file

The template copied earlier is set up to run the transpiled `main.ts` file when your action is triggered. The same as running `node main.js`.

By default the `main.ts` has an example asynchronous function `wait`. This is a good example of ***Separation of Concerns***, notice how the `wait` function is completely decoupled from the action. Below is the contents of the `main.ts` file with the `run` function highlighted.

```typescript/5-12
// main.ts
import * as core from '@actions/core'
import {wait} from './wait'

async function run(): Promise<void> {
  try {
    const ms: string = core.getInput('milliseconds')
    core.debug(`Waiting ${ms} milliseconds ...`)

    core.debug(new Date().toTimeString())
    await wait(parseInt(ms, 10))
    core.debug(new Date().toTimeString())

    core.setOutput('time', new Date().toTimeString())
  } catch (error) {
    core.setFailed(error.message)
  }
}

run()
```

The aim is to replace this `wait` function with the logic to list all branches that are 90 days old.

## What is my Output?

Before writing a line of code, answer the following questions:

- What is my output? What is my goal?
- Which APIs can get me that output?
- What input do I need for those APIs to get the required output?

For most Actions, the answer to the first question, "What is my output?" is usually a notification of some kind. In this case I opt for creating a new GitHub issue as creating an issue and tagging people will send them an email. The issue will be populated with old branches so I also need this information in there.

I almost always have the [GitHub API documentation](https://developer.github.com/v3/) and the [Octokit](https://octokit.github.io/rest.js/v18) open in a browser tab because I refer to it really frequently when making Actions. I identify that the three APIs I need are:

- [List branches](https://developer.github.com/v3/repos/branches/#list-branches) which returns all branches for the repo.
- [Get Branch](https://developer.github.com/v3/repos/branches/#get-a-branch) which gives me the name of the last person that committed to the branch and the date that was.
- [Create Issue](https://developer.github.com/v3/issues/#create-an-issue) I'm sure you can guess what that does :) 

## Mocking the Issues Endpoint

The following method aims to have as little boilerplate code in tests as possible. It focuses on producing the output with as little dependencies as possible. 

The first step is to create a test that checks an issue has been created. As the API we need the handy npm library, [@actions/github](https://www.npmjs.com/package/@actions/github) that wraps the GitHub API and is tailored to work well with actions.

The below test creates the `octokit` object and spies on the `octokit.issues.create` function. This allows us to later check this function was called in our code and validate that we've got our expected output.

```typescript
// __tests__/old-branches-issue-create.test.ts
import * as GitHub from '@actions/github'
import {
  OctokitResponse,
  IssuesCreateResponseData
} from '@octokit/types'

const octokit = GitHub.getOctokit('fakeToken')
let createIssueSpy: jest.SpyInstance
let issuesCreateResponse = {} as OctokitResponse<IssuesCreateResponseData>

beforeEach(async () => {
  createIssueSpy = jest
    .spyOn(octokit.issues, 'create')
    .mockResolvedValue(issuesCreateResponse)
})

```

## Creating an Empty Function

Create a new empty function in a new file that will contain the logic only. It's going to accept the `octokit` object which is a wrapper for the GitHub API. The tests will spy on the functions we are going to use.

```typescript
// src/old-branches-issue-create.ts
import { Octokit } from '@octokit/core/dist-types'
import { RestEndpointMethods } from '@octokit/plugin-rest-endpoint-methods/dist-types/generated/method-types'

export function oldBranchesIssueCreate(octokit: Octokit & RestEndpointMethods) {
}
```

We finally have some code 🥳. Now is the time to plug that into your `main.ts` file. Remember we are ***not** going to test the `main.ts` file. This file will have the least logic in possible and is only tested when we have completed the coding of the action.

```typescript
// main.ts
import * as GitHub from '@actions/github'
import { oldBranchesIssueCreate } from './old-branches-issue-create'

async function run(): Promise<void> {
  try {
    const octokit = GitHub.getOctokit(process.env.GITHUB_SECRET)
    await oldBranchesIssueCreate(octokit)
  } catch (error) {
    core.setFailed(error.message)
  }
}

run()
```

## Test the Create Issue Spy is Getting Called

Write the test below which passes the `octokit` object with the spy configured to listen for calls to the `octokit.issues.create` function. I've added `...` to represent the code we've already written.

```typescript
// __tests__/old-branches-issue-create.test.ts
import { oldBranchesIssueCreate } from '../src/old-branches-issue-create'

...

test('expect an issue to be created if includes old branches', async () => {
  await oldBranchesIssueCreate(octokit)
  expect(createIssueSpy).toHaveBeenCalled()
})
```

## Call the Create Issue Spy in your Code.

The above test will fail as we only wrote an empty function. Implement the logic in your code. Update your code until the test passes.

```typescript/5
// src/old-branches-issue-create.ts
import { Octokit } from '@octokit/core/dist-types'
import { RestEndpointMethods } from '@octokit/plugin-rest-endpoint-methods/dist-types/generated/method-types'

export async function oldBranchesIssueCreate(octokit: Octokit & RestEndpointMethods) {
  await octokit.issues.create()
}
```
## Add Owner, Repo and Title to the Create Request 

So far so good--but this code won't actually work when deployed. The `create` function requires a few more details to work. The [Octokit documentation](https://octokit.github.io/rest.js/v18#issues-create) shows us that the following are required to create an issue:
- owner
- repo
- title

You can get the `owner` and the `repo` from the GitHub `context` populated from the event. The title will be whatever title you want the issue to have. 

The below code sets up a spy on the `context` object to return the repo and owner. This is populated by the Github Action pipeline.

```typescript/5-12,17-19
// __tests__/old-branches-issue-create.test.ts
import { oldBranchesIssueCreate } from '../src/old-branches-issue-create'
import {Context} from '@actions/github/lib/context'
let context: Context

// Context coming from github action
beforeEach(() => {
  context = {
    repo: {
      repo: 'github-guru',
      owner: 'peterjgrainger'
    }
  } as Context
})

test('expect an issue to be created if includes old branches', async () => {
  await oldBranchesIssueCreate(octokit, context)
  expect(createIssueSpy).toHaveBeenCalledWith({
    repo: 'github-guru',
    owner: 'peterjgrainger',
    title: 'List of Old Branches'
  })
})
```

Update the code to make this test pass

```typescript/3,5-9
// src/old-branches-issue-create.ts
import { Octokit } from '@octokit/core/dist-types'
import { RestEndpointMethods } from '@octokit/plugin-rest-endpoint-methods/dist-types/generated/method-types'
import {Context} from '@actions/github/lib/context'

export async function oldBranchesIssueCreate(octokit: Octokit & RestEndpointMethods, context: Context) {
  await octokit.issues.create({
    repo: context.repo.repo,
    owner: context.repo.owner,
    title: 'List of Old Branches'
  })
}
```

<mark>Pro Tip: use the spread operator `...context.repo` instead of enumerating the `repo` and `owner`</mark>. Below is an example of what I mean.

```typescript
{
  ...context.repo,
  title: 'List of Old Branches'
}
```

Now we can create an issue 🎉. However, it's not very useful--it has no content. Next we will call the list branches and get branches API to figure out which branches are old.

## Get a List of All Branches.

I'll go over this a little quicker than the `Create Issue` spy in the previous section. The steps for testing are very similar.

The function to get all branches is detailed in the [octokit docs](https://octokit.github.io/rest.js/v18#repos-list-branches). It's under `ocktokit.repos.listBranches`, which takes `repo` and `owner` as arguments. Below is the test to check list branches has the correct arguments.

```typescript
// __tests__/old-branches-issue-create.test.ts
import * as GitHub from '@actions/github'
import { oldBranchesIssueCreate } from '../src/old-branches-issue-create'

import {
  ReposListBranchesResponseData,
  OctokitResponse,
  ReposGetBranchResponseData,
  IssuesCreateResponseData
} from '@octokit/types'

let listBranchesResponse: OctokitResponse<ReposListBranchesResponseData>
let getBranchResponse: OctokitResponse<ReposGetBranchResponseData>
const octokit = GitHub.getOctokit('fakeToken')

beforeEach(() => {
  listBranchesData = [
    {
      name: 'master'
    }
  ] as ReposListBranchesResponseData

  listBranchesResponse = {
    data: listBranchesData
  } as OctokitResponse<ReposListBranchesResponseData>

  listBranchesSpy = jest
    .spyOn(octokit.repos, 'listBranches')
    .mockResolvedValue(listBranchesResponse)
})

test('expect list branches to be called', async () => {
  await oldBranchesIssueCreate(octokit, context)
  expect(listBranchesSpy).toHaveBeenCalledWith({
    owner: 'peterjgrainger',
    repo: 'github-guru'  
  })
})

```

The most important part of this is the use of `as ReposListBranchesResponseData` and `as OctokitResponse<ReposListBranchesResponseData>` which lets us create a typed object and populate it with only the parameters that we need.

Write the code that makes the tests pass. To do this, call the API to list branches then call the API to create an issue.

```typescript/6-8
// src/old-branches-issue-create.ts
import { Octokit } from '@octokit/core/dist-types'
import { RestEndpointMethods } from '@octokit/plugin-rest-endpoint-methods/dist-types/generated/method-types'
import {Context} from '@actions/github/lib/context'

export async function oldBranchesIssueCreate(octokit: Octokit & RestEndpointMethods, context: Context) {
  await octokit.repos.listBranches({
    ...context.repo
  })

  await octokit.issues.create({
    repo: context.repo.repo,
    owner: context.repo.owner,
    title: 'List of Old Branches'
  })
}
```

## The Get Branch Function

The last step is to call the [Get Branch](https://octokit.github.io/rest.js/v18#repos-get-branch) function for each branch to find out when and who altered the branch last.

This is done in a similar way to `createIssue` and `listBranches` so I'm not going to go through this part in depth like the other functions. The `getBranch` function can be accessed through the `octokit` object using the function `octokit.repos.getBranch`.

## The Full Solution

Whether you read the whole post or skipped to this section it's useful to see my test setup in full. The following code creates two tests. The first one checks that `listBranches` was called and the second that and issue was created with the title `Old branches <today's date>`. The body of the issue is a list of all the old branches along with who was the last committer on that branch.

```typescript
// __tests__/old-branches-issue-create.test.ts
import { oldBranchesIssueCreate } from '../src/old-branches-issue-create'
import * as GitHub from '@actions/github'
import { Context } from '@actions/github/lib/context'
import {
  ReposListBranchesResponseData,
  OctokitResponse,
  ReposGetBranchResponseData,
  IssuesCreateResponseData
} from '@octokit/types'

let context: Context
let listBranchesData: ReposListBranchesResponseData
let listBranchesResponse: OctokitResponse<ReposListBranchesResponseData>

let getBranchResponse: OctokitResponse<ReposGetBranchResponseData>
let getBranchData: ReposGetBranchResponseData

let issuesCreateResponse: OctokitResponse<IssuesCreateResponseData>

let listBranchesSpy: jest.SpyInstance
let getBranchesSpy: jest.SpyInstance
let createIssueSpy: jest.SpyInstance
const octokit = GitHub.getOctokit('fakeToken')
let getInput: (parameter: string) => string
let excludedAuthor: string

// Context coming from github action
beforeEach(() => {
  context = {
    repo: {
      repo: 'github-guru',
      owner: 'peterjgrainger'
    }
  } as Context
})

// Set up API responses
beforeEach(() => {
  listBranchesData = [
    {
      name: 'master'
    }
  ] as ReposListBranchesResponseData

  listBranchesResponse = {
    data: listBranchesData
  } as OctokitResponse<ReposListBranchesResponseData>

  getBranchData = {
    name: 'master',
    protected: false,
    commit: {
      author: {
        login: 'peterjgrainger'
      },
      commit: {
        author: {
          name: 'Peter Grainger',
          date: '2012-03-06T15:06:50-08:00'
        }
      }
    }
  } as ReposGetBranchResponseData

  getBranchResponse = {
    data: getBranchData
  } as OctokitResponse<ReposGetBranchResponseData>

  issuesCreateResponse = {} as OctokitResponse<IssuesCreateResponseData>
})

// Set API responses in mocks
beforeEach(async () => {
  listBranchesSpy = jest
    .spyOn(octokit.repos, 'listBranches')
    .mockResolvedValue(listBranchesResponse)
  getBranchesSpy = jest
    .spyOn(octokit.repos, 'getBranch')
    .mockResolvedValue(getBranchResponse)
  createIssueSpy = jest
    .spyOn(octokit.issues, 'create')
    .mockResolvedValue(issuesCreateResponse)
})

test('expect list branches to be called', async () => {
  await oldBranchesIssueCreate(octokit, context)
  expect(listBranchesSpy).toHaveBeenCalledWith({
    protected: false,
    owner: 'some-owner',
    repo: 'some-repo',
    per_page: 100
  })
})

test('expect an issue to be created if includes old branches', async () => {
  await oldBranchesIssueCreate(octokit, context)
  expect(createIssueSpy).toHaveBeenCalledWith({
    repo: 'some-repo',
    owner: 'some-owner',
    title: 'Old branches ' + new Date().toString().slice(0, 15),
    body:
      '## Branches older than 30 days\nmaster: last commit by @peterjgrainger'
  })
})

```

## Testing can be Easy and Fun

I think most people dislike testing because it's seen as a chore. Tests are seen as something that __needs__ to be done, like brushing your teeth, stopping you from doing something fun. I hope I've showed you that testing can be frustration free and as enjoyable as writing the code itself.

The easiest way to make tests less complicated is organise your code. Passing mocks or spies into the function rather than using imports stops the testing code becoming complex and hard to modify. Planning your expected outputs will keep you mind focused on your goal.

I always thought that my code should never change to make tests easier. But it's the other way around, making tests easier produces better code. [Kent C. Beck](https://en.wikipedia.org/wiki/Kent_Beck) has written some great books on this subject and is a great person to follow if you are interested in more tips like this.
