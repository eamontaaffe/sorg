+++
published_at = 2019-12-11T01:27:24Z
title = "Exploring Actions"
+++

![Sumo strong eggnog](/assets/images/nanoglyphs/005-actions/eggnog@2x.jpg)

This is Nanoglyph, a newsletter about sustainable software. It ships weekly, though like many software projects, its release date has been known to slip. It's still very much in alpha. If you'd like to get it in your inbox, you can [subscribe here](https://nanoglyph-signup.herokuapp.com).

---

This week involved the fulfillment of an old Heroku tradition by making a batch of [sumo strong eggnog](https://github.com/seaofclouds/sumostrong/blob/master/views/eggnog.md). It's a convenient recipe in that you can buy the heavy cream and whole milk involved in cartons of exactly the right size so there’s none leftover. It makes enough to fill three [Kilner clip top bottles](https://www.amazon.com/Kilner-Square-Clip-Bottle-34-Fl/dp/B005N984I8/) almost perfectly. If the idea of strong eggnog is even remotely appealing, consider it a strong recommendation.

The usual format is three links from the week with some commentary, but to keep things dynamic, I’m playing with the format to instead talk about a small project in a little more depth -- like a push version of a blog post.

---

## Migrating to Actions (#github-actions)

A few weeks ago I spent some time migrating the program that generates my blog (and this newsletter) over to use [GitHub Actions](https://github.com/features/actions). My CI compiles and runs a test suite like you'd expect, but goes a little beyond that to [push the deployment](/aws-intrinsic-static) that puts it on S3.

It’s automated to the degree that making live changes only requires pushing to `master` or merging a pull request. That's been a huge advantage over the years as [almost 50 people](https://github.com/brandur/sorg/graphs/contributors) have sent me patches to fix typos and inaccuracies. Wherever I am -- in a meeting, commuting, on a run -- I hit the "Merge" button from my phone, and a few minutes later the build is live.

### Commodifying CI (#commodifying-ci)

For my money, Travis was one of the most important service innovations of the decade. With a little help from GitHub, they made setting up CI so easy that it didn't make sense _not_ to do it. Even pushing a repo with 50 lines of code that you never intend to look at again, you may as well add a few lines to `.travis.yml` and get CI running. It might just be that little bit of assistance needed by your future self to fix a project they remember practically nothing about. Even if the CI doesn’t do anything elaborate, `.travis.yml` still serves as a codified reference for how to build the project and run tests.

More recently, Travis [was acquired](https://news.ycombinator.com/item?id=18978251), and based off the buyer and the subsequent attrition of engineering staff, it’s hard to imagine that terms were favorable. Things are still running more or less the same, but given the sheer expense that must be involved in doing free builds for a sizable part of the world’s open source software, some of us have been looking for alternatives, just in case.

GitHub’s Actions was a timely arrival. Although officially described in grandiose terms that can leave the average reader a little confused as to what the product does, they can be appropriately thought of as Travis, with a dose of containers mixed in. Actions describe jobs that will run on repository events like a push, opened pull request, or cron — perfect for CI, and useful beyond that too.

The steps of a job can be defined as traditional shell commands, but also as Docker containers to run. In my own recipe I have shell steps like:

``` yml
- name: Install
  run: make install
```

Intermingled with containers like:

``` yml
- name: Install Go
  uses: actions/setup-go@v1
  with:
    go-version: 1.13.x
```

The path in `uses` refers to a GitHub repository, so the code above refers to the [actions](https://github.com/actions) organization in GitHub, which contains a number of common containers. Versioning is possible with a number of familiar Git mechanisms:

``` yml
steps:    
  - uses: actions/setup-node@74bc508
  - uses: actions/setup-node@v1
  - uses: actions/setup-node@v1.2
  - uses: actions/setup-node@master
```

Steps can reference Docker Hub with the magic `docker://` prefix:

``` yml
- name: My first step
  uses: docker://alpine:3.8
```

---

### The container as unit of modularity (#container-modularity)

This leads into Actions’ most important innovation: the container as a unit of workflow modularity. (That might sound dangerously platitudinous, but hear me out.)

Containers have always been modular in that containers reference other containers during builds, and that modularity's been one of their key selling points since Docker's first release. The difference with Actions is that the modularity is taken a step further in making containers the convention for encapsulating reusable code — whether it’s cloning a Git repository, setting up a Go environment, or deploying to a specific service like AWS, the Actions container allows complex functionality to be reused easily in a generic workflow.

This is interesting because despite the popularity of containers, many services have been pushing in a much different direction: JavaScript. If you use AWS Lambda, or any of its numerous clones (Twilio Functions, etc.), workflows are written in JavaScript and the unit of reuse is a copy/pasted JS blob, or Node package.

As someone who believes that humanity can do better than JavaScript, this is exciting. If I want to write a package for use with GitHub actions I can do so using the widely understood convention of containers, and in a language that's well-designed and type-safe.

---

### What Actions gets right (#right)

Containers are a nice touch. In Travis, you could get some code reuse by manually pulling down scripts and running them, but it was overly difficult and haphazard. A single, prescribed system that provides built-in modularity is a huge step forward.

Also, acknowledging that builds are really just a series of steps, and that it’s not necessary to differentiate the category of step like setup vs. build, is a simplification that works. Jobs in Actions look like this:

``` yml
steps:
  - name: Step 1
  - name: Step 2
  - name: Step 3
```

Travis differentiated phases with `install`, `script`, `before_script`, `after_success`. It wasn't a robust abstraction:

* The ordering of phases wasn’t intuitive, so you’d have to look up the job lifecycle every time.

* Even with the plethora of phases, you’d eventually have to start chaining commands within one of them (usually `script`). Travis allowed separate steps with a YAML array, but made no qualms if any of the failed, so users have to either `set -e` or chain commands with `&&` to get the behavior they wanted.

Avoiding &&-chaining allows the UI to be improved considerably. Steps have names, and success/failure status, run time, and build output is cleanly assigned to each one. Troubleshooting failed builds gets much faster.

![Steps in GitHub Actions](/assets/images/nanoglyphs/005-actions/steps@2x.png)

Steps can be configured individually using `with` to specify parameters for containers (e.g. to specify the version of Go or Postgres to install) or `env` to specify step-specific variables. This is good because it lets you see which particular steps need specific variables instead of mixing everything into a global env. Explicit always beats implicit.

``` yml
- name: "Create database: sorg-test"
  run: createdb sorg-test
  env:
    PGHOST: localhost
    PGPORT: ${{ job.services.postgres.ports[5432] }}
    PGUSER: postgres
    PGPASSWORD: postgres
    PGDATABASE: postgres
```

---

### Where Actions is the same (#same)

While containers _seem_ to be an elegant idea, interacting with them isn't always straightforward. e.g. GitHub provides a straightforward recipe for getting containerized Postgres up and running as a service in the background. But once you have a Postgres server going, you want to interact with it, and that necessitates command line tooling like `createdb` and `psql`. Those utilities are happily installed inside the container, but that's not much use to the Actions recipe looking in from beyond the fence.

The easiest thing to do is to fall back to the versions available with `apt-get` -- just like with Travis. That's easy to do, but given that builds leverage Ubuntu LTS releases, versions available tend to be tragically out of date — and not by months, by years.

And the same goes for any command line utility. I use ImageMagick for image resizing. You can get a container version of it, but passing arguments and files into that is awkward, and invoking it once per operation is inefficient and difficult to do from another script/executable. I fell back to the one in `apt-get`, and while it worked well enough, it was ancient to the degree that CLI usage has since changed and more modern formats like <acronym title="High Efficiency Image Coding">HEIC</acronym> were not supported.

It’s not easy to see how GitHub would go about solving this one, but doing so would give them a huge leg up over what Travis was ever able to do. Imagine for a moment a GitHub-run package manager agnostic to the underlying OS (say in the spirit of Homebrew) which provided recent versions for a wide range of key utilities.

---

While technology is moving so quickly that we get new products and services around the clock, it's not that often that we see one that's novel and useful to a hugely broad audience [1]. GitHub Actions is both, and to a greater degree than we've seen from anything else in years.

As usual, you can [sign up here](https://nanoglyph-signup.herokuapp.com) to get next week's edition send to your mailbox. And feel free to hit the reply button to tell me whether this deep dive (medium dive?) did anything for you.

Until next week.

[1] The absolute ascendancy of GitHub in the developer community means that this product will likely be used by the majority of developers over the coming years.
