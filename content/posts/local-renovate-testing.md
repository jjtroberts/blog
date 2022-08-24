---
title: "Local Renovate Testing"
date: 2022-08-24T08:25:59-05:00
draft: false
type: post
image: "/images/renovation.jpeg"
---

# Renovation
Following the openscap post in this series on local security testing, I'd like to focus on
a tool called [Renovate](https://docs.renovatebot.com/). And no, this isn't about redoing
your master bath.

Renovate is an opensource tool that automates dependency management. It can be ran as an npm
package, a container image, or as a free GitHub App. While it has a plethora of built-in datasources
it can leverage to track updates (rpms, pypi, github-releases, etc.) it also supports modules/plugins
so you can write custom code to support your edge cases.

## Use Case
Platform One's Iron Bank heavily relies on Renovate to provide continuous dependency updates across
approximately 2,000 repositories. The tool runs nightly (but can be executed as often as is necessary for your requirements) and opens pull requests when it detects new versions of the upstream packages used in
your project. The PRs can be automerged and dependencies can be pinned to minor releases, grouped, or excluded altogether.

The benefit in having this git-native workflow is that each PR triggers our CI process which lints, builds and scans; providing a tight feedback loop for engineers. We have another bot that runs multiple times a day and scans for PRs with no accompanying issue. If it detects a PR opened by `renovate-bot`, then it creates a new issue linked to the PR and labels it so that it appears in our backlog.

## The Problem
Renovate is not an easy tool to use if you've never encountered it before. The documentation doesn't provide enough examples, and I haven't been able to find many helpful posts online giving solid details on how other people have implemented the tool.

Our engineers struggle at times to implement a working `renovate.json` configuration, which can leave gaps in our dependency update automation that isn't discovered until much later. Our hosted renovate tool will only check the development branch of any repo that has the `renovate-bot` user. So whenever we want to test changes we have to merge our feature branch into the development branch, and iterate or rollback our changes. Needless to say that is not a process that leads to developer happiness. So I started looking at how to provide my team with a local solution using Gitea, MySQL, Docker Compose and Renovate.

## The Solution
Keep in mind that this is a work in progress and heavily leans toward my work with repo1.dso.mil and registry1.dso.mil. You can see what I came up with here: https://github.com/jjtroberts/renovate-test

The goal is to have a local `docker-compose` environment that stands up gitea/mysql/renovate so that we could commit our code changes to gitea and, while having renovate running alongside, also be able to execute renovate externally with the custom ironbank renovate module (included in the Iron Bank renovate image) as often as we wanted. This would enable a fast feedback loop and allow us to iterate over config changes in our renovate.json without polluting the repo1 project with our commits on the development branch. It would also reduce noise in terms of Gitlab MRs created as we would no longer be executing against a live version control system tied to issue backlogs.

Admittedly, renovate does not need to be included in `docker-compose.yaml` because we primarily execute it using `make run` as a standalone docker container attached to the network created by docker compose. However, it is helpful to have a running renovate container if you should want to attach to it and investigate its configuration for learnings. It has been helpful to have this setup when comparing different versions of renovate when chasing down bugs in new releases.

For example, when a new release of upstream renovate breaks functionality within our custom ironbank module. We actually had this issue not too long ago where the renovate image containing the ironbank module was updated by renovate to the latest release. This happens all the time, however we didn't realize right away that renovate was no longer creating pull requests. We traced the bug back to code that had been added to the ironbank module that no longer played nice with renovate and were able to fix. However, without a local environment to test these different versions we would not have easily identified which code commits within our custom ironbank module had caused the issue.

Check out the [repo](https://github.com/jjtroberts/renovate-test), test it out and help me make it better.