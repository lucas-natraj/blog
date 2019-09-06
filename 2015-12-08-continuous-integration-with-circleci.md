---
layout: post
category : ci
author: Lucas Natraj
title: Continuous Integration with CircleCI
---

We've been using [CircleCI](https://circleci.com) for continuous integration builds and tests. CircleCI is a fantastic low-cost-of-entry solution when using one of their supported languages and test runners, but for C# .NET code, the amount of customization required is significant. A large amount of time was spent attempting to create the correct configuration in the `circle.yml` file that would install mono, DNVM, DNX, and any required libraries. This became more and more specific to the CircleCI machines as time went on — trying to understand what was installed on them already, what version of linux they ran, etc.

In the end, we decided that a more flexible approach, though requiring more initial setup, was to run our tests themselves in a docker container. The artemis-tests container contains all the unit and integration tests. It connects to the other infrastructure and artemis containers through docker for integration tests, and it has the source code for running unit tests. This image is based on the same microsoft/aspnet docker image that the artemis image itself is based on, so it has the same internal container environment as the production image has.

As a result of this approach, the CircleCI-specific script (`circle.yml`) has only a handful of lines in it, thus making our CI build more easily portable to another provider, if needed. Additionally, the same setup as used in the CI build can be deployed and run locally with exactly the same results.