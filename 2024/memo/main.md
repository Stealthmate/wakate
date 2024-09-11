# Learning about monorepos, microservices and teams the hard way

| Repo / Problem | Artifacts            | Project             | State    |
|----------------|----------------------|---------------------|----------|
| Monorepo       | External             | Internal / External | External |
| Polyrepo       | Internal / External  | External            | External |

3 years ago I inherited a generic web app project in my company. It was a typical pair of back-end and front-end, spread across 2 git repositories, with zero CI. Bonus points for having to deploy manually by logging to production with SSH, running `git pull` and then rerunning commands in a tmux session.

Fast forward a year from that. Now we are running CI jobs to run lint and tests, build docker images and push them to the internal container registry. Deployment is still done via SSH, but now we run `docker-compose` from inside CircleCI. Progress!

A few months later I learned about Kubernetes and microservices, and I guess you can see where this is going. I set up the Kubernetes cluster, a GitOps repo and ArgoCD, and now we also have CD pipeline. Yet more progress!

Fast forward to the present and now we have around 10 different apps running on that cluster, most of which are some premature type of microservice. And since they were all part of the same "application" as before, all the code lives in a single repo - the Monorepo.

---

As I said, it's been 3 years since I started working on that project, and after spending countless hours thinking about what is the most logical way to organize and manage things, I reached some hopefully rational conclusions. So I decided to put all of this experience in writing, in the hopes of someone somewhere sometime finding it helpful.

## Monorepo vs Polyrepo

After reading a bunch of blog posts, reddit threads and stack exchange questions I realized this is probably the Git equivalent of Emacs vs Vim. Or maybe not. The point is that people have varying opinions on this, and sometimes (oftentimes?) I feel they're talking about completely different problems.

So here's my hot take on the subject.

### What problem does a source code repository solve in the first place?

This seems like an easy question, but the answer is actually surprisingly non-simple. And the reason is that current popular platforms like GitHub, GitLab and others, have a lot of popular features that solve problems completely unrelated to source code management.

The original goal of, and biggest reason to use, a repository is to store source code in a way that's efficient and useful for developers to easily track and update. I don't think anyone would disagree with that. But is it the only thing a repository can do?

#### The Repository as an artifact store

I bet that at some point most developers end up having to download some kind of binary, jar, python wheel, etc. from a GitHub release. Have you ever thought about what that really means? What's the difference between, let's say installing from PyPI vs `curl`-ing a wheel from a GitHub release?

In my eyes, there's not much difference. Sure, GitHub releases are tied to a certain commit tag, which gives them a bit more credibility maybe? I'm not denying that there are valid reasons to choose PyPI over GitHub (or vice versa), but as a package __consumer__, as long as you trust the provider, you don't really care about whether you're installing from PyPI, a GitHub release, a random FTP server or whatever. And that's because, at least conceptually, they all implement the same interface, namely:

- Upload an artifact with a specific identifier (including version)
- Download an artifact, given its identifier (including version)

So in a sense, you can, and a lot of people __do__, use GitHub as an artifact store. In fact, you can "emulate" PyPI in GitHub - just create a bunch of GitHub repositories for each file, create a text file and create releases for all versions by tagging commits which change the text file to the name of the version.

#### The Repository as a project management platform

Again, I bet most developers have opened a pull request, commented on an issue, referenced some other pull request or issue in the comment, etc. And most of those have probably done code review on a PR, merged it and closed a relevant issue.

All of this is very nice functionality, but it solves a completely different problem from source code storage. Theoretically, in an alternate universe GitHub would only provide git API access, and all issues, pull requests etc. would be managed in JIRA, which would have brilliant integration with GitHub. Quite the fantasy, I know, but in theory it's possible, and that theory serves to show that we're talking about different problems.

#### The Repository as a definition of state

I'd say the concept of GitOps is pretty popular these days and I personally use ArgoCD for my deployments, so I'm going to base my opinion on that.

What is the purpose of a GitOps repository? I'd say it is to represent some required state of __The Real World__. For example ArgoCD is basically a tool that tries to force __The Real World__ into the state defined by the manifests stored in the GitOps repository.

---

So, given all of the above insight, how should you choose between Monorepo and Polyrepo? Well in my opinion, you should think very carefully about how you're solving the above problems first. The choice of Monorepo vs Polyrepo is a trade-off between varying levels of handling the above use cases. Some thing are easier to do in a monorepo, others are easier to do in a polyrepo. But **picking a repository style is not a solution by itself**.

Next I'm going to explore how the two approaches deal with each of the above use cases.

### The Polyrepo approach

First of all, I'm going to state an important assumption.

In a poly repo approach, each __application unit__ gets its own repository. I'm intentionally using this weird self-invented term, because I want it to mean something very specific.

>> A __application unit__ is something that _may_ produce multiple artifacts, but they all live under a _single version_.

Read this twice. A single repository should only ever produce one version of a thing, but it's okay to produce multiple artifacts.

The reason I'm making this assumption is because, if you have two "different" things living under the same repository, it's really a monorepo of 2 things. And in my experience, the difference between 1 and 2 is the most important.

So, with this in mind, let's see what happens if you roll polyrepo.

#### Storing artifacts

Given that a single repository holds only one application unit, it becomes possible to use the Git platform as an artifact store. You can setup your CI pipeline to automatically push a GitHub release with the relevant artifacts on each new version tag.

This works because of our assumption from before. The _thing_ living inside of the repository is a single unit, managed by a single version. So the repository version is effectively the same as the version of the unit.

To give you a not-so-simple example, imagine you're developing a REST API. You decide to release v1.0 and so you `git tag -a v1.0` your current commit. Now your CI produces a few artifacts, for example:

1. A docker image `my-api:v1.0`
2. An OpenAPI spec `my-api-v1.0.yaml`

Finally, you push those artifacts to a GitHub release tied to the tag `v1.0` and call it day. Whenever someone wants to use `v1.0` of your API, they can always go to GitHub and download both the source code _and_ the relevant artifacts. Hooray!

On a side note, you may be thinking "Why push the image to GitHub instead of Docker Hub/ECR/etc?" You can indeed do that, but the point of this example is to illustrate that _you can choose not to_. In the case of a docker image, maybe it's indeed better to use Docker Hub. But what if this was a private Python package? What if it was a JAR? What if it was a Protobuf file?

If you have the option of using an external artifact store, and if you think that using it is better than GitHub - please do as you see fit. But if you're using GitHub, whether because of choice or lack thereof, you should remember that _it only works when you have a single application unit_.

#### Project management

