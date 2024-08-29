# Monorepo vs Polyrepo - What problems do they actually solve?

3 years ago I inherited a generic web app project in my company. It was a typical pair of back-end and front-end, spread across 2 git repositories, with zero CI. Bonus points for having to deploy manually by logging to production with SSH, running `git pull` and then rerunning commands in a tmux session.

I spent 3 years working on this thing and I after spending I-don't-know-how-many hours researching I-don't-know-how-many topics, I finally reached a point where I feel comfortable making a choice between monorepo and polyrepo. So I decide to put all of my thoughts down into writing, so that someone somehwere sometime can hopefully find it helpful.

## Modern git repositories are not just source code repositories

Let me ask you a question:

> What problem does a git repository solve?

It seems like an easy question, but the answer is actually surprisingly non-simple. And the reason is that current popular platforms like GitHub, GitLab and others, have a lot of popular features that solve problems completely unrelated to source code management.

The original goal of, and biggest reason to use, a repository is to store source code in a way that's efficient and useful for developers to easily track and update. I don't think anyone would disagree with that. But is it the only thing a repository can do?

### The Repository as an artifact store

I bet that at some point most developers end up having to download some kind of binary, jar, python wheel, etc. from a GitHub release. Have you ever thought about what that really means? What's the difference between, let's say installing from PyPI vs `curl`-ing a wheel from a GitHub release?

In my eyes, there's not much difference. Sure, GitHub releases are tied to a certain commit tag, which gives them a bit more credibility maybe? I'm not denying that there are valid reasons to choose PyPI over GitHub (or vice versa), but as a package __consumer__, as long as you trust the provider, you don't really care about whether you're installing from PyPI, a GitHub release, a random FTP server or whatever. And that's because, at least conceptually, they all implement the same interface, namely:

- Upload an artifact with a specific identifier (including version)
- Download an artifact, given its identifier (including version)

So in a sense, you can, and a lot of people __do__, use GitHub as an artifact store. In fact, you can "emulate" PyPI in GitHub - just create a bunch of GitHub repositories for each file, create a text file and create releases for all versions by tagging commits which change the text file to the name of the version.

### The Repository as a project management platform

Again, I bet most developers have opened a pull request, commented on an issue, referenced some other pull request or issue in the comment, etc. And most of those have probably done code review on a PR, merged it and closed a relevant issue.

All of this is very nice functionality, but it solves a completely different problem from source code storage. Theoretically, in an alternate universe GitHub would only provide git API access, and all issues, pull requests etc. would be managed in JIRA, which would have brilliant integration with GitHub. Quite the fantasy, I know, but in theory it's possible, and that theory serves to show that we're talking about different problems.

#### The Repository as a definition of state

I'd say the concept of GitOps is pretty popular these days and I personally use ArgoCD for my deployments, so I'm going to base my opinion on that.

What is the purpose of a GitOps repository? I'd say it is to represent some required state of __The Real World__. For example ArgoCD is basically a tool that tries to force __The Real World__ into the state defined by the manifests stored in the GitOps repository.

---

So, given all of the above insight, how should you choose between Monorepo and Polyrepo? Well in my opinion, you should think very carefully about how you're solving the above problems first. The choice of Monorepo vs Polyrepo is a trade-off between varying levels of handling the above use cases. Some thing are easier to do in a monorepo, others are easier to do in a polyrepo. But **picking a repository style is not a solution by itself**.

## Software units and Compositions

Before we talk about whether to have one or many repositories, we first should think about what we're putting inside of them. Again, let me pose you a question:

> What should a git repository contain?

"The source code!" I hear you exclaim. But I want you to think about _what_ source code that is. For example, the source code for [requests](https://github.com/psf/requests), a popular python library is the sole inhabitant of its own repository. On the other hand, the canonical python implementation, [CPython](https://github.com/python/cpython), lives inside a repository alongside the Python standard library and other things.

Now, here's the interesting part: **would you call the CPython repository a monorepo?**

I honestly cannot predict if/how the answers will split. But what I can say is that, whether the CPython repository is a monorepo or not, it still represents _one concept_ - the canonical Python implementation.

As a maths person, I like putting my definitions first, so I want to take the time to define two very specific concepts here, which will heavily influence the monorepo vs polyrepo debate to follow.

### The Software unit

Consider `requests` libary. It comes as a single Python package, it has a well-defined version number and can be considered more or less as a single, inseparable entity. Those are some interesting and, as it turns out, key properties when considering how a repository can be used. So, in good old programmer fashion, let's abstract them away!

> A _**software unit**_ is an inseparable piece of software that produces one or more artifacts, which are tied to a single version number.

Note that, while there is only one version number, a software unit can produce multiple artifacts. For example `requests` ships both a source distribution (sdist) and a wheel. Another example could be your generic REST server - we could produce both a docker image that runs the API, alognside an OpenAPI JSON file. The important part is that all artifacts are tied to the same version number.

The other important property of a unit is _inseparability_. Basically this means that it doesn't make sense to consider only part of the software unit. Barring some extra special exceptions, generally you wouldn't want to install only "part of" `requests`, and you wouldn't want to run only "part of" a single REST server.

### The Composition

Most big software will probably not fit into the definition of a software unit. For example, the [Ubuntu](https://ubuntu.com/) Linux distribution does have a single version number, but it can hardly be considered inseparable. Amazon's AWS platform (as far as I know) doesn't even have a version and it is definitely not inseparable.

But both of these examples can still be considered as a _whole_. They have a lot of parts, but those parts still do combine in order to form a single entity. So, again in good old programmer fashion, let's turn that into an abstraction.

> A _**composition**_ is a group of one or more software units and/or other compositions that, taken together, provide some kind of unique value.

Note that, given this definition, `requests` also doubles as a composition. In fact, every software unit can, and most likely is, considered a composition of 1. After all, why even bother giving a version to something that doesn't provide any kind of value?

On the other hand, a composition does not necessarily need to contain it's parts' source code. For example the Ubuntu repository (as far as I know) doesn't contain the source code for all the packages that Ubuntu bundles as part of their install, but only specifies their versions.

But the point I'm trying to make is to be aware of the difference between these two concepts. Most business software is composed of a lot of moving parts, some of which are compositions themselves, and treating them the same as a unit would quickly lead to problems.

### Units, compositions and repositories

So why did I bother defining these two terms? Because I want to convince you that the **difference between monorepo and polyrepo basically boils down to which one you put in the repository**.

In a monorepo, you have multiple units in the single repository, but they (should) still form a single composition. Google is famous for having a monorepo with literally all of their code inside a single repository, but that can still be considered a single composition - the Google ecosystem (or something along those lines). On the other hand, putting the CPython source code in the same repository as the Linux kernel wouldn't make for a good composition, because they don't really combine into something that can be considered whole.

In contrast, in a polyrepo approach you can have either a single unit or a composition per repository. I say _either_, because if you want eventually provide some kind of composition to your users, you _will_ have at least one repository that serves to combine all your units into one. For example, most packages that Ubuntu uses probably have their own repositories, but the Ubuntu repository is the one that hosts the _composition_ that is Ubuntu itself.

When you think about it like this, the question of "monorepo vs polyrepo" starts becoming a bit hazy. The problem is not whether to have one repository or many, but rather **whether each repository contains only one unit/composition or multiple units/compositions**. This is a somewhat trivial, but important distinction, and you will see why in a moment.

So, armed with all of these definitions, let's finally start analyzing the specifics about monorepos and polyrepos!

## Monorepo vs Polyrepo

Phew. Okay, we're finally at the most important part of this post. Since I've already made you wait long enough, I'm going to put the conclusion at the top:

> The choice between monorepo and polyrepo is **about what tools you use to organize the source code and _not_ about how you develop your application.**

Read it twice, then read it one more time. If you take anything from this post, I really want it to be this idea.

Choosing whether to have one or more repositories, with one or more units/compositions in them, is **not** about microservices, monoliths, shared dependencies, versioning schemes and other problems that relate to coding, building and running an application. Instead, it's about **deciding what job you want your VCS platform to fulfill**. And that in turn is **influenced by what features the platform provides**.

I know that this sounds abstract and hand-wavy, so I'm going to give you a few examples of how monorepos and polyrepos differ in some specific areas. I'm also going to give you a few examples of problems that exist regardless of whether you pick a monorepo or polyrepo, and to be honest I think understanding those problems is even more important than the choice itself.

### Example #1: Working with the source code

In order to actually do any development, first you need to download the code to your own PC. Then you open the files in an IDE, do some edits, save, maybe run some tests and then push your changes back to the server. Simple enough.

This is an area where monorepo vs polyrepo does have an influence, but probably not in the way you might initially think.

#### Download Size

In a polyrepo setup, each repository is small enough, which is good. But in a monorepo, a single repository could be very big. So even simply `git pull`-ing the repository could take, to put it mildly, an inconvenient amount of time.

_But is that a fair comparison?_

If we are talking about downloading all of your company's source code, you could just as easily write a 10-line script to `git pull` every repository under your company in the same root directory. And that would be at least equally as slow as `git pull`-ing one repository with all the code in it.

On the other hand, if you only want the code for one unit or composition, do you really need to pull the whole monorepo? You could instead use something like [sparse checkouts](https://stackoverflow.com/a/63786181) to pull only part of the repository, which doesn't really differ that much from pulling a single small repository in a polyrepo setup.

Now you might say:

> But the whole point is that `git pull` is easier than `git sparse-checkout` and I don't even know what the latter is!

And I agree with that sentiment, which is exactly why I want to stress that **this problem is a result of using `git`**. In other words, it has nothing to do with how you write your code, and everything to do with how `git` does/does not work. Or to put it in other words - if git sparse checkouts were something everybody knew, or if there was a popular tool that could easily clone parts of a repository, the repository size wouldn't be a reason to choose polyrepo over monorepo and vice versa.

#### Access permissions

I think this is one of the few, but major factors when deciding between monorepo and polyrepo, because _hiding information_ is something that, at least to my knowledge, is generally not possible inside the same repository. In this sense, if you *want to* manage write and especially read access to certain files, you are more or less forced to adopt a polyrepo approach.

However.

If you think about this a bit deeper, it is, again, a problem *tooling*. Theoretically speaking, if there was a platform called MyAwesomeGit that allowed you to set permissions on whether people can checkout/push specific files and directories, you could happily use that alongside a monorepo. So, again, **this problem is also a result of using a certain tool**.

### Example #2: Project organization

Earlier I mentioned that modern git platforms are actually more than a simple VCS. The biggest example of this is supporting issues, pull request threads, kanbans and other *project management features*. So you can expect that monorepos and polyrepos will take (or not take) advantage of these features differently.

#### With a monorepo

Consider a monorepo first. As we defined earlier, the idea behind a monorepo is that you keep your whole composition inside a single repository. If you put the monorepo on GitHub, it allows you to create issues that span multiple units, and then push and merge PRs that modify those units in the same place, at the same time. A lot of people would call these merges *atomic commits*.

Now you might expect me to talk about how atomic commits are nice because you can modify everything in a single go or whatnot, but that would actually be an example of something that is irrelevant to the type of repository, as outlined by [^1].

Instead, I want you to focus more on the fact that issues, pull requests, etc are stored in a single place. Depending on the scale of your repository, this can mean one of two things:

- Tracking code changes for new feature development and bugfixes is easy, cross-referencing issues is easy, grouping issues into projects, kanbans and so on is also easy, therefore the approach is awesome!
- You have 100 different projects (still within the same repository) in progress at the same time, maybe 5k open issues at any given moment, 500 people are working on the codebase and it's virtually impossible to find what you're looking for on GitHub.

Do you see where I'm getting at? Depending on scale, you will probably either love having everything in one place or hate it. If you're in the first category, then good for you. But if you're in the second, and if you isolate the problem (project management and organization), then you realize that **the problem is a result of using a certain tool** (in this case GitHub). Maybe you realize that GitHub doesn't cut it and decide to switch to [~~JIRA~~](https://ifuckinghatejira.com/), [ClickUp](https://clickup.com/), [Trello](https://www.trello.com), [Asana](https://www.asana.com) or whatever your tool of choice is. And you would probably be right to do so. But the point is that if you manage to solve the organization problem, you can happily keep using your monorepo.

#### With a polyrepo

Now consider the polyrepo approach. You have N repositories on GitHub, each of which manages its own issues, PRs, projects and so on. If you're already using one of the external tools above, then you don't really have a problem. But if you are using GitHub for project management, then now you end up in either of these categories:

- Everything is nicely separated, each repository corresponds to a project, they all get to manage their own issues and code changes independently and everyone worked happily ever after.
- [Displaying the birthday date on the settings page](https://www.youtube.com/watch?v=y8OnoxKotPQ) takes 6 months because the relevant code is spread across 20 different repositories, each managed by a separate team, PRs take 1 week to review and nobody has any idea where the feature is tracked in the first place.

Obviously the latter is an exaggeration, but the point is that, at least in theory, even if you fall in the second category you can still solve the problem without switching to a monorepo. For example, if, in an alternate universe, there existed a blazingly fast [JIRA](https://ifuckinghatejira.com/) which could give you an overview of every repository you had and automatically link PRs across repositories so that they're opened and merged at the same time, you could have an experience similar to that of a monorepo.

So, yet again, **the problem is a result of using a certain tool**.

### Example #3: Artifact storage

This is actually quite a limited and specific thing, but I wanted to put it here because I think in general it is a decent idea.

As we discussed earlier, you can use GitHub and the likes as a simple artifact store. But in order to do that (properly anyway), you need the guarantee that *a single repository hosts one thing, that is managed by one single version*. Sound familiar? It's the software unit!

Of course, you're not *required* to use your VCS provider as an artifact store, but if you wanted to do that then you are pretty much required to put put units inside their own repositories, which automatically implies taking the polyrepo approach.

But do not get confused - using a monorepo does not necessarily mean that all your software units no longer have independent versions! I honestly cannot stress this enough, so I'm going to explicitly discuss it as an example of a bad reason to choose a monorepo vs a polyrepo.

### Bad Example #1: Version Management

TODO

### Bad Example #2: TODO

### Bad Example #3: TODO

## It really just boils down to tooling


## References

- [^1] https://www.snellman.net/blog/archive/2021-07-21-monorepo-atomic/