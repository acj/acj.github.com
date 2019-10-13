---
layout: post
title: The robots are coming for your maintainership
---

I recently got access to GitHub Actions through the beta program, and it got me thinking about what parts of my life as both a hobbyist and professional programmer could be automated. Adding continuous integration to a handful of side-project repositories was an easy one. For something juicier, I wondered if it would be possible to automate the often tedious process of updating packages for Linux distributions. I'm the maintainer of a couple of packages for Alpine Linux, and it would be nice to let the machines do the boring parts.

After thinking for a bit, it seemed possible. There were several problems to solve:

* Monitor for new releases. I want my packages to stay current, and that means I have to remember to check the upstream repositories for updates. That's fragile and annoying.
* If there's a new release, fetch and build the commit for it
* If the build succeeds, create a branch that bumps the package's version number (and therefore the source commit) to match the new release
* Notify me if there's a new version or a problem with the build

## Let's build it

For the impatient, the final workflow is [here](https://github.com/acj/aports/blob/a1de66fcb0c90e74d3b5953577be3f6a575bcd3e/.github/workflows/bcc.yaml). I'll go through each sub-problem and highlight the part of the workflow that solves it.

If you're not familiar with GitHub Actions yet, I recommend taking a quick look at the [core concepts](https://help.github.com/en/articles/about-github-actions#core-concepts-for-github-actions) documentation so that words like "workflow", "step", and "action" will have the right meaning.

### Monitor for new releases

My hope was that Actions could run code in my repository in response to an event in a _different_ repository. For example, whenever a new `release` event is triggered on the [iovisor/bcc](https://github.com/iovisor/bcc) repository, I'd like an action to run in one of my personal repositories, such as a fork of the upstream one. Unfortunately, that doesn't seem possible -- at least for now -- without having an external mechanism to trigger the action. (I'd love to be wrong about this.)

The backup plan was to use Actions' support for cron jobs. That is, you can [schedule](https://help.github.com/en/articles/workflow-syntax-for-github-actions#onschedule) an action to run periodically without any other input or trigger. Using a schedule instead of an event trigger introduces other challenges, but it's reliable and simple to set up, and it's the solution that I decided to use.

```yaml
on:
  push:
    branches:
    - actions/abuild/*
  schedule:
    - cron:  '0 9 * * *'
```

In addition to running the action every day at 9am UTC, I configured the action to run every time I push commits to any branch whose name matches `actions/abuild/*`. This makes testing and experimenting _much_ easier. It's important to note, however, that cron jobs only run for the master branch.

Among the challenges of using a scheduled action is the fact that, on most days, there will be nothing to do. The workflow will need to recognize that there hasn't been a new release and terminate the build to conserve CPU time -- but without marking the build as failed. Actions provides an [if](https://help.github.com/en/articles/workflow-syntax-for-github-actions#jobsjob_idstepsif) derivative that's very helpful here.

A related challenge surfaces on the days _after_ a new release has shipped. It can take days or even weeks to get a new package version reviewed and merged upstream, and in the meantime the scheduled action continues to run each morning. To avoid generating errors or failed builds during that time period, the workflow has to recognize that a branch already exists for the new version. Solving this problem wasn't too painful and led to a second action that I'll discuss later.

### Fetch and build the commit for a new release

To actually relieve the maintenance burden, my workflow needs to fetch the new version of bcc and build it as closely as possible to the way that the Alpine project's continuous integration system works. It would be a bummer to do all of this work only to have the new branch fail in Alpine's CI. Luckily, the Alpine developer tooling is pretty good and includes a tool called [abuild](https://github.com/alpinelinux/abuild) that can do most of the heavy lifting. The first piece of this puzzle is [action-abuild](https://github.com/acj/action-abuild), an action that wraps the `abuild` tool and takes care of details like package signing.

If anything goes wrong during this stage, there's usually an issue with the Alpine-specific patches that are applied before bcc is compiled, and I'll need to manually fix them. Automation can't solve that problem just yet.

This stage comprises three related steps: figure out which version we want, check out the repository, and then try to build the new version. Note the use of `echo ::set-output ...`, which is a [clever mechanism](https://help.github.com/en/articles/development-tools-for-github-actions#set-an-output-parameter-set-output) for returning outputs from bash- and Dockerfile-based actions.

```yaml
    - name: Resolve package and release versions
      id: resolve_versions
      env:
        GITHUB_REPO: iovisor/bcc
        PACKAGE_PATH: community/bcc
      run: |
        PACKAGE_VERSION=$(curl -s https://raw.githubusercontent.com/alpinelinux/aports/master/$PACKAGE_PATH/APKBUILD | grep "pkgver=" | sed -E 's/pkgver=//g')
        RELEASE_VERSION=$(curl -s https://github.com/$GITHUB_REPO/releases.atom | grep "<title>" | grep -G -o "v[^ <]\+" | head -1 | tr -d 'v')
        if [ "$RELEASE_VERSION" == "$PACKAGE_VERSION" ]; then
          echo ::set-output name=have_new_version::false
        else
          echo ::set-output name=have_new_version::true
          echo ::set-output name=package_path::"$PACKAGE_PATH"
          echo ::set-output name=package_version::"$PACKAGE_VERSION"
          echo ::set-output name=release_version::"$RELEASE_VERSION"
          echo ::set-output name=branch_name::"$PACKAGE_PATH-to-$RELEASE_VERSION"
          echo ::set-output name=commit_message::"$PACKAGE_PATH: update to $RELEASE_VERSION"
        fi
    - name: Check out aports
      if: steps.resolve_versions.outputs.have_new_version == 'true'
      uses: actions/checkout@master
      with:
        fetch-depth: 1
    - name: Try building the new release version
      if: steps.resolve_versions.outputs.have_new_version == 'true'
      uses: acj/action-abuild@v1.0.0
      with:
        PACKAGE_PATH: ${{ steps.resolve_versions.outputs.package_path }}
        PACKAGE_VERSION: ${{ steps.resolve_versions.outputs.package_version }}
        RELEASE_VERSION: ${{ steps.resolve_versions.outputs.release_version }}
```

### If the build succeeds, create a branch with an updated version number

If we get this far, it means that a new version of bcc has been released, and `action-abuild` has successfully built a package from it. This is great news. All that's left is to create a branch with the changes and notify me so that I can do a final check and submit the package upstream.

I created a second action called [action-branch-from-working-copy](https://github.com/acj/action-branch-from-working-copy) to handle this. This action surfaces parameters like `branch_name`, `commit_message`, and `commit_author_name` so that the resulting branch and commit are customizable. Some repositories, such as Alpine's [aports](https://github.com/alpinelinux/aports), ask package maintainers to use a consistent style in their commit messages (e.g. "community/bcc: update to 0.11.0"), and these parameters make it possible to automate all of that.

As I mentioned before, a few things can happen at this stage. If the new branch (whose name is derived from the version number, e.g. `bcc-to-0.11.0`) already exists in the repository, then we have to decide whether to treat this as a success or a failure. Which result is appropriate arguably depends on the workflow that's invoking the action, and so I've added a `fail_if_branch_exists` input parameter. There's a corresponding output parameter called `branch_name_already_exists` so that the calling workflow knows whether a successful result means that the action created a new branch or that it did nothing (it's already there, so you're good to go).

One lingering question is what to do if the working copy doesn't contain any changes. The action currently returns a successful exit code in that case. Feedback is welcome, as are PRs.

```yaml
    - name: Create a branch with updated package version
      id: create_branch
      if: steps.resolve_versions.outputs.have_new_version == 'true'
      uses: acj/action-branch-from-working-copy@v1.0.0
      with:
        BRANCH_NAME: ${{ steps.resolve_versions.outputs.branch_name }}
        COMMIT_MESSAGE: ${{ steps.resolve_versions.outputs.commit_message }}
        COMMIT_AUTHOR_NAME: 'Adam Jensen'
        COMMIT_AUTHOR_EMAIL: 'acjensen@gmail.com'
        FAIL_IF_BRANCH_EXISTS: 'false'
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Notify me if there's a new version or a problem with the build

We've nearly done it. The new version of bcc has been successfully built, a branch has been created with the updated version number, and it's ready to be submitted upstream. But how do these robots inform me that any of this has happened?

I've chosen Slack as the notification mechanism, but there are many options. If you wanted to take this workflow a step further, it could even create the upstream PR for you. For now, I'm just happy that I didn't need to fetch the new code, build it, and juggle the version numbers.

```yaml
    - name: Notify Slack
      if: steps.resolve_versions.outputs.have_new_version == 'true'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      run: |
        branch_name="${{ steps.resolve_versions.outputs.branch_name }}"
        branch_already_exists=${{ steps.create_branch.outputs.branch_name_already_exists == 'true' }}
        if [ "$branch_already_exists" == "true" ]; then
          curl -s -X POST -H 'Content-type: application/json' --data "{\"text\":\"New version of ${{ github.workflow }} is available, but branch '$branch_name' already exists\"}" $SLACK_WEBHOOK_URL
        else
          PR_URL="https://github.com/acj/aports/pull/new/$branch_name"
          curl -s -X POST -H 'Content-type: application/json' --data "{\"text\":\"New branch for ${{ github.workflow }} created: $PR_URL\"}" $SLACK_WEBHOOK_URL
        fi
```

Note that this step doesn't handle failure cases. If a previous step failed, I'll get an email from GitHub saying so.

## Wrapping up

We did it! And I'm happy to report that this thing has already saved me time and toil. The 0.11.0 release last week was the first real test of my workflow, and -- naturally -- bcc made a couple of bigger changes that broke the Alpine build. The important thing is that I learned about the build failure immediately and had an upstream PR merged within 72 hours. With a little luck, the next release will build cleanly, and I'll wake up to a new branch that I can test and publish.

GitHub Actions is a game changer. It's not the first workflow engine (far from it), and as far as I can tell it's not enabling anything that we couldn't already do, but it'll be the first workflow engine that a lot of folks use. Many won't even realize that they're using one. When you grab an action and have it running on your repository a few seconds later, it feels like magic. For example, I added a simple build-test-report CI action to a personal Go project repository recently. The only "configuration" was choosing which `go test` arguments to use. The whole thing took less than five minutes, which included several test builds. It shows the true power of this ecosystem -- forums, good documentation, easy feedback/remixing, containers, and event-driven architecture.

And what probably matters more than anything else is that it was _fun_ to build this. That's huge.