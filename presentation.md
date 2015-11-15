![fill](boston.jpg)

# [fit] npm install
### where it's complicated
### and how it works

---

## **Rebecca Turner**

### CLI Engineer at
### ![inline 50%](npm-logo.png)

### Twitter: **@ReBeccaOrg**
### Github, IRC: **iarna**

^ Hi, I'm Rebecca Turner and I'm an engineer working on npm. I'm the one mostly responsible for npm@3.

---

![100%](npm-logo.png)

^ npm is lots of things. It's a company (that I work for <3), it's a website, it's a registry, it's a self-hosted server product, and it's a command line tool. That last one is where I spend my days, so that's what you get to hear about.

^ The complexity in the command line tool comes in part from just being a successful, several year old open source project. It's gained a lot of features and a lot of them were added without a view to the larger architecture. But some of the complexity is a necessary result of what it does. And it's that latter kind of complexity I want to talk to you all about today.

---

# Package Managers

^ So packages managers. They're a thing. Today we're going to build a package manager. Or rather, we're going to design one.

---

# What's'it do?

mnp ==  manage node packages

^ This is gonna be a node package manager. Unlike npm, we can make it stand for something, let's say "manage node packages".

---

# What's'it do?

Install packages. Duh.

`mnp install https://xyz-collective/xyzzy-1.0.0.tgz`

^ Obviously its main thing is installing packages. But let's frame that as a user need…

^ The great folks at the XYZ collective have released a new version of their xyzzy framework and I want to use it in my project.

---

# Install libraries

1. Download a thinger.
2. Put the thinger in my `node_modules`.

^ So that sounds easy, we just download it and extract it where node can see it.

---

# Downloading

* Proxy servers
* Unreliable connections
* Buggy servers
* Easily overloaded firewalls

^ Downloading sounds simple but is potentially fraught. I'm gonna hand wave this, but you could do a whole talk on just how downloading reliably is complicated.

---

# Packages

1. require('xyzzy') == node_modules/xyzzy/
2. node_modules/xyzzy/package.json (?)

^ What _are_ packages exactly? For our purposes, they're a folder that goes in your `node_modules` folder and optionally they can have a `package.json`

---

# `package.json`

`require('xyzzy')` == `xyzzy/index.js`

unless xyzzy/package.json:
`{main: 'lib/elsewhere.js'}`

then:
`require('xyzzy')` == `xyzzy/lib/elsewhere.js`

^ When you `require` a module, node has to figure out what javascript to load. It uses `index.js` unless you have a `package.json` with a `main` key, in which case it will load that as your module. This is actually the only use of your `package.json` that node itself makes.

---

# Install libraries

^ Downloading a thing and extracting it somewhere is simple enough that it could be a shell script. In fact, I've worked at companies where this was the package management solution.

^ Once you've got a way of downloading and extracting packages, you're going to almost immediately want to keep track of what packages you have installed…

---

# Dependency Lists

## package.json

```json
    {
      "main": "index.js",
      "mnp-dependencies": [
        "https://xyz-collective/xyzzy-1.0.0.tgz"
      ]
    }
```

^ And hey, node has this `package.json` thing, let's stick our dependency list in there...

---

# Dependency Lists

```sh
mnp install
```

^ And with that, we can just run `mnp install` and have it install all of our dependencies.

---

# Install libraries

1. For each thinger in our dependencies…
2. Download a thinger.
2. Put the thinger in my `node_modules`.

^ Ok, that didn't make it much more complicated and it sure is more useful... but we probably don't want it downloading packages than once…

---

# Install libraries

1. For each thinger in our dependencies *that isn't already in our node_modules*…
2. Download a thinger.
2. Put the thinger in my `node_modules`.

^ That's better, now we aren't downloading everything every time.

^ But now that we have dependency lists, just everyone's gonna want to use them. Sooner or later xyzzy is gonna ship with its own dependencies…

---

# Install libraries

1. For each thinger in our dependencies that isn't already in our node_modules…
2. Download a thinger.
2. Put the thinger in my `node_modules`.
4. Go into each thinger's folder and run `mnp install`

^ That's easily handled, for each dependency we extract we run `mnp install` again on it.

^ So now we've got a very basic package manager. This is basically how many package managers work. Let's take a moment and think a bit about how we model our dependencies, and how our package manager does.

---

# Modeling
## As a Tree

```
our-app
└── xyzzy
    ├── foo
    │   ├── bar
    │   └── frob
    └── bar
```

^ The most direct way to model our project dependencies as a tree. One thing requires another which requires yet more…

^ It's also kind of neat that node let's us store our packages in this same shape.

^ But, but, but, Rebecca… wouldn't this be better thought of as a directed graph? That's fair, as you can have circular dependencies, that is, when a package depends on something that then depends on that first package. I'm going to hand wave that for now– it's rare, but it does happen in the real world occasionally. Depending on the nature of the circle, you often just have to declare the packages broken.

---

# Modeling
## In isolation

```
foo:
  path: our-app/node_modules/xyzzy/node_modules/foo
  dependencies: [bar, frob]
```

^ But the package manager we've designed sees these in isolation, without understanding its context in the larger tree. It knows the thing we're installing, where we're installing it, and what it requires, but that's it.

^ Now, knowing a little thing about how node's `require` works, we can make a simple refinement to this. As we've already established, `require` looks in the current package's `node_modules` folder. But it ALSO looks up the tree to any dependencies the higher levels. This means we can add naive but still useful deduplication…

---

# Modeling
## With minimal context

```
foo:
  alreadyinstalled: [bar, xyzzy]
  path: our-app/node_modules/xyzzy/node_modules/foo
  dependencies: [bar, frob]
```

^ By tracking what's already be installed by our parents, we can avoid installing a copy at this level. This here is essentially how npm prior to version 3 understood the tree.

---

# Install libraries

1. For each thinger in our dependencies that isn't already in our node_modules and isn't in alreadyinstalled in our model.
2. Download the thinger.
3. Put the thinger in my `node_modules`.
4. Go into each thinger's folder and call `mnpInstall()`

^ So let's add that back to our design...

---

# Modeling
## Concurrency

```
xyzzy:
  alreadyinstalled: []
  path: our-app/node_modules/xyzzy
  dependencies: [foo, bar]
```

^ Because everything in node is async by default, the natural thing to do is to kick off all of our dependencies downloads and installs at the same time.

^ Indeed, this is what npm has done for a long time. This works great in that it can really use all of your available resources and get stuff done in as little wall clock time as possible.

---

# Install libraries
## DO ALL THE THINGS

1. For all our dependencies that aren't already in our node_modules and aren't in alreadyinstalled in our model, do at once:
  ⓐ Download a thinger.
  ⓑ Put the thinger in my `node_modules`.
  ⓒ Go into each thinger's folder and call `mnpInstall()`

^ This means most everything will try to download all at once. Mostly this works ok, but on shaky network connections or if you're behind an overloaded proxy or firewall then you may get intermittent errors from the networking layer.

---

# HOLD UP THERE!
# FEATURE REQUEST
## C Modules
### Pls add this now, kthx

^ Wait wait, this package manager is fine and all, but what it really needs is to support modules written in C.

---

# Install libraries

1. For all our dependencies that aren't already in our node_modules and aren't in alreadyinstalled in our model, do at once:
  ⓐ Download a thinger.
  ⓑ Put the thinger in my `node_modules`.
  ⓒ Go into each thinger's folder and call `mnpInstall()`
2. Compile ourselves

^ Oookay, so how do we do that? You can't really compile stuff in advance and just distribute binaries 'cause we don't know what OS our users will be using. So that means we need to compile at install time...

---

# C modules

## package.json

```json
    {
      "main": "index.js",
      "mnp-scripts": {
        "install": "node-gyp rebuild"
      }
      "mnp-dependencies": [
        "https://xyz-collective/xyzzy-1.0.0.tgz"
      ]
    }
```

^ So let's add some config that let's you run a compiler at install time. The usual way to compile C modules for node is to use `node-gyp`.

---

# OH NO!
#Race conditions!

^ Unfortunately, there's another word for concurrency: Race conditions. And we just introduced one. Much of the bug fixing in late npm v1 and early npm v2 was centered around taming them.

^ So let's explore exactly what our race condition _is_ and then we'll look at how to fix it.

---

# Dependency tree

```
our-app
├── bar
└── xyzzy
    ├── foo
    │   ├── bar
    │   └── frob
    └── bar
```

^ We're going to say that our app relies on the `bar` utility module, just like almost everyone else.

^ The installer is going to start by running its install steps for both `bar` and `xyzzy` at the same time.

---

# Trace the installer's actions

⒈ Add `bar` and `xyzzy` to `alreadyinstalled`.
⒉ Start download for `bar` and `xyzzy`.

^ Start the download of all of the dependencies.

---

# Trace the installer's actions

⒈ Add `bar` and `xyzzy` to `alreadyinstalled`.
⒉ Start download for `bar` and `xyzzy`.
⒊ Download of `bar` finishes; extract into `node_modules`.
⒋ Download of `xyzzy` finishes; extract into `node_modules`

^ The downloads finish so start them both extracting.

---

# Trace the installer's actions

⒈ Add `bar` and `xyzzy` to `alreadyinstalled`.
⒉ Start download for `bar` and `xyzzy`.
⒊ Download of `bar` finishes; extract into `node_modules`.
⒋ Download of `xyzzy` finishes; extract into `node_modules`
⒌ Extract of `bar` finishes; go into it and run `mnpInstall()`
➱ ⓐ No deps for `bar`; run compile step.

^ `bar` finishes extracting first, run mnpInstall for it. Since it has no deps, we run it's compile step. Turns out it's a C library so that'll take a bit…

---

# Trace the installer's actions

⒈ Add `bar` and `xyzzy` to `alreadyinstalled`.
⒉ Start download for `bar` and `xyzzy`.
⒊ Download of `bar` finishes; extract into `node_modules`.
⒋ Download of `xyzzy` finishes; extract into `node_modules`
⒌ Extract of `bar` finishes; go into it and run `mnpInstall()`
➱ ⓐ No deps for `bar`; run compile step.
⒍ Extract of `xyzzy` finishes; go into it and run `mnpInstall()`
⒎ (elided: install `foo` `frob`)

^ Meanwhile `xyzzy` finishes extracting– let's go run `mnpInstall()` in it.  It has two deps, but one is `bar`, which is installing at the top level so we skip it. The other is `foo`– you know the drill, so I'm gonna skip all the actions. What's important is that it finishes installing `foo` and `frob` before `bar` has finished compiling…

---

# Trace the installer's actions

⒈ Add `bar` and `xyzzy` to `alreadyinstalled`.
⒉ Start download for `bar` and `xyzzy`.
⒊ Download of `bar` finishes; extract into `node_modules`.
⒋ Download of `xyzzy` finishes; extract into `node_modules`
⒌ Extract of `bar` finishes; go into it and run `mnpInstall()`
➱ ⓐ No deps for `bar`; run compile step.
⒍ Extract of `xyzzy` finishes; go into it and run `mnpInstall()`
⒎ (elided: install `foo` `frob`)
⒏ Compile `xyzzy`; EXPLODES

^ So since `xyxxy`'s direct dependencies are done installing, we go and run its compile step. But OH NO, it uses `bar` as part of it's own compilation and it errors out because `bar` isn't done compiling yet.

---

# Fixing our race condition

^ At first you might be tempted to throw a lock on the compiles, so that only one runs at a time. This isn't a bad idea, because you don't really want dozens of simultaneous C compilers running. But while it fixes the problem we just walked through, if `bar` was waiting in the download or extract steps, it'd still explode. This is basically what npm 1 and 2 did, and in practice it only exploded VERY rarely, mostly because very few node modules depend on other node modules during their compile step, combined with it requiring a very specific set of dependencies and some real bad luck when it comes to timing.

^ No, fixing this isn't so easy, any option is going to require some not insubstantial increase to what the installer knows. One fairly straightforward approach would be to make the `alreadyinstalled` list be promises that a resolved after their compile step finishes– so then `xyzzy` would wait for that promise to resolve before running its own compile step. Another approach, and the one npm 3 took, is a little more complicated on the surface but helps structurally eliminate race conditions. This means that race conditions like these don't require special consideration.

---

# npm scripts

## prepublish, preinstall, install, postinstall, preuninstall, uninstall, postuninstall, preversion, version, postversion, test

^ With just an `install` script, we ran into a race condition. npm runs a few more scripts during installs, as you can see, which is part of why a more comprehensive answer to race conditions was preferable.

---

# Remodeling the install

## Multi-stage installation

1. Figure out everything we're gonna do.
2. For each kind of thing we're gonna do one at a time.

^ So at it's heart, it's eeeasy, just figure out what we're going to do before doing anything. Then do all the things. This is for `mnp` of course, `npm` has a few more steps. So let's fill that out a little…

---

# Remodeling the install

## Multi-stage installation

1. Record what's currently installed.
2. Make an in-memory tree of what we'd like to be installed.
3. Compare the two and make a list of things to do.
4. For each kind of thing we're gonna do one at a time.

^ Read what's currently installed. Figure out what we'd like everything to look like after we finish running.

^ Compare those two views and make a list actions to take to change the first into the second.

^ And then do all the things. Uh, but what are the things?

---

# Remodeling the install

## Multi-stage installation

4. For each kind of thing we're gonna do one at a time.
  ⓐ Download all the things
  ⓑ Extract all the things
  ⓒ Compile all the things

^ Well, the same stuff we were doing before, downloading, extracting and compiling. But instead of possibly running these steps all at the same time for different modules, we're running all of one step TOGETHER.

^ So some of these things can be done concurrently like downloading and extracting. By contrast, compiling needs to happen strictly in order, such that one's dependencies are all installed before we start.

^ We could do this by just making installs promises and resolving all the promises of our dependencies before starting a compile. This would work, but it's really hard to inspect. An easier to inspect (and thus debug[!!]) approach is to make a list in advance and sort that list. Then we can just install in that order, nice and linear. We can ALSO print that list out.

^ Incidentally, if you run an npm@3 install with --loglevel=silly you can see the before and after trees, and the list of steps we're going to execute. One of the guiding principles of npm 3 has been to be transparent about what the installer is thinking.

---

## Multi-stage installation

1. Record what's currently installed.
2. Make an in-memory tree of what we'd like to be installed.
  ⓐ Download all the NEW things and look inside them to see what they require.
3. Compare the two and make a list of things to do.
4. For each kind of thing we're gonna do one at a time.
  ⓐ Extract all the things
  ⓑ Compile all the things

^ Well, that last slide was a bit of a lie, 'cause while it's what we'd like to do, right now the only way to learn about a module is to download it and read its `package.json`.

---

# Modeling
## As a Tree

```
our-app
└── xyzzy
    ├── foo
    │   ├── bar
    │   └── frob
    └── bar
```

^ One of the cool things about moving to this approach is that the installer sees the dependency tree the same way we do. This makes debugging much easier, because any time the installer's model doesn't match ours that's a bug in either it or us.

---

# HOLD UP THERE!
# FEATURE REQUEST
## Version Ranges
### Pls add this now, kthx

^ Y'know what would be real nice? Not having to update the URLs for packages eeeevery time there's a little patch release.

---

# Version Ranges

    ^3.5.0
    >= 2.0.0 < 2.3.0
    ~1.7.3

^ So this is where semver ranges come in. But this is ALSO where a registry comes in. There needs to be some place we can ask "hey, what versions exist of xyzzy and what's their download URL".

^ This is where a registry comes in. I'm gonna handwave the registry, but it has it's own complexity.

---

# Registries & Publication

^ Now that we have registries we have to have some sort of publication, where we tell the registry about a new version. I'm not gonna go into detail on that, but there is one feature of publication that I want to make note of.

---

# dist-tags

```sh
mnp publish
mnp publish --tag beta
mnp dist-tags add xyzzy@1.3.7 beta

mnp install xyzzy
mnp install xyzzy@beta
```

^ Oh no, this has led to another new feature! So publishing is obvious enough right? But it's really handy to be able to specify beta version or other named versions somehow.

---

# dist-tags

```sh
mnp publish # same as: mnp install --tag latest
mnp publish --tag beta
mnp dist-tags add xyzzy@1.3.7 beta

mnp install xyzzy # Same as: mnp install xyzzy@latest
mnp install xyzzy@beta
```

^ And if we're gonna allow betas, we also need to make sure that a regular install doesn't get them. So we have a default tag of latest.

^ When you go to install, if you don't ask for a version, you get latest.

---

# Recording Versioned Dependencies

```json
{
  "main": "index.js",
  "mnp-scripts": {
    "install": "node-gyp rebuild"
  }
  "mnp-dependencies": {
    "xyzzy": "^1.0.0"
  }
}
```

^ By changing our dependency list over to an object we can easily express our dependence on any 1.x version of xyzzy.

---

# Matching Versions

## >= 1.0.0 && < 2.0.0

^ So we're asking for the most recent version that's semver compatible. Nooow, if the beta is a part of the 1.x series then it _is_ semver compatible, but we probably don't want it.

---

# Matching Versions

## Fetch list of tags:
## `{latest: '1.3.6', beta: '1.3.7'}`
## >= 1.0.0 && <= 1.3.6

^ So now we fetch the list of tags and take the tag version into account in our range.

^ Now, node-semver has one more restriction that complicates this a bit. It has a rule that says if the version has a prerelease tag it shouldn't match a range specifier, unless that range specifier is for the same major/minor/patch.

---

# Prerelease Versions

## 1.3.5-a.1
## ^1.3.5-0 == >= 1.3.5-0 && < 2.0.0

^ A prerelease specifier is the part after a hyphen in a version. You can ask for them to be included by specifically including a pre-release specifier in your version range.

---

# Matching Versions

## Fetch list of tags:
## [fit] `{latest: '1.3.7-a.3', beta: '1.3.7'}`
## [fit] >= 1.0.0 && <= 1.3.7 AND NOT prerelease

^ So this means that if latest is a prerelease it won't be selected if you ask to install a semver range.

---

# Matching Versions

## [fit] `{latest: '1.3.7-a.3', beta: '1.3.7'}`
## [fit] `mnp install xyzzy # installs 1.3.7-a.3`

^ This is why it's important that installing without a version should install _latest_ and not _*_. In fact, until this past thursday npm was installing _*_, which meant you wouldn't get latest if it was a prerelease, which was super confusing.

---

# Multistage Steps

1. Record what's currently installed.
2. Make an in-memory tree of what we'd like to be installed.
   ⓐ Get the metadata for all the things as we do this.
3. Compare the two and make a list of things to do.
4. For each kind of thing we're gonna do one at a time.
   **ⓐ Download some of the things**
   ⓑ Extract all the things
   ⓒ Compile all the things

^ So interestingly, what this let us do is delay downloading modules that have metadata on the registry until later. Of course, not for everything, we still have the old style dependencies that we have to download and extract a `package.json` from to know what they do.

---

# HOLD UP THERE!
# FEATURE REQUEST
## Bundled Dependencies!!
### Pls add this now, kthx

^ So maybe you depend on 10,000 small modules and want to speed up installs for your users. Or maybe you want them to use the version you developed with? Or maybe you're the package manager itself and you need to be installable without existing oneself? Bundled dependencies are one way to solve this.

^ The interesting thing about this as a feature is that it really doesn't change the high level steps we take during installation…

---

# Bundled Dependencies

### Get the metadata for all the things as we do this.

^ Bundled dependencies do make metadata collection harder, because we need the metadata for the module in question, PLUS for every bundled dependency.

---

# Bundled Dependencies

## Version Conflicts

^ It also raises some tricky new design decisions. Like what do we do if I bundle a dep with a version that's not compatible with my package.json? Do we keep the version you bundled? Or declare it invalid and replace it? npm has answered this question both ways at different times.

^ The other really tricky bit here for npm is that currently the registry doesn't keep track of bundled dependency metadata, which means it has to extract the tarballs to get it, which is one of the reasons npm 3 is slower than npm 2.

---

# HOLD UP THERE!
# FEATURE REQUEST
## Installing from a local file
### Pls add this now, kthx

^ Heeeey, while we're here, could we maybe install directly from files on disk? I've got this thing and I don't want to upload it somewhere to use it.

^ Sure, that seems pretty easy, "downloading" becomes "uhm, look over here at this thing we already have".

---

# Local Dependencies

```json
{
  "xyzzy": "file:///path/to/xyzzy.tar.gz"
  "xyzzy": "file:relative/path/xyzzy.tar.gz"
  "xyzzy": "xyzzy.tar.gz"
}
```

^ So what do local file dependencies look like? Well, the most obvious place to start is file URLs. But what if you want it to be relative to your package? Uh... well, we can do a bogus fake file url. It's not valid, but we can make it work. (Technically it should be file://relative but no one would read it correctly.) Or finally would could just take a filename.

^ The filename looks pretty appealing but it does mean we have to have some heuristics, because we have to distinguish it from a tag. The way npm does this, incidentally, is to check to see if it exists on disk, if it does it treats it as a file, otherwise it's a tag.

---

# HOLD UP THERE!
# FEATURE REQUEST
## Installing from a local *folder*
### Pls add this now, kthx

^ We support local files? Well, what about a local directory?

^ We can do that! Specifiers can look the same as as with local files too!!

---

# Local Directory Dependencies

```
mkdir xyzzy
mnp install xyzzy
```

^ Awesome, now you have more ambiguity to resolve! If I have a folder named xyzzy do I install that? Or do I install the version from the registry?

^ The best answer here might be to simply refuse to run, because we can't understand the user's intent. If they said `xyzzy/` then we KNOW they meant a folder and if they said `xyzzy@latest` we KNOW they meant the registry.

^ npm, incidentally, will treat this case as an install of the _directory_ and not a registry package. This probably is convenient sometimes, but because most people don't use local dependencies, I think it's more often confusing.

---

# Local Directory Dependencies
## Building Packages

^ We've not discussed at all how packages are made, but if we're going to support local directory dependencies, we're going to have to actually make packages out of those directories. Now, for `mnp` as we've defined it so far, that's probably just making a tarball. For `npm` this is a whole host of new complexity, as it requires that you have dependencies and dev dependencies installed and package authors can hook this to do build steps prior to making that package tarball.

---

# HOLD UP THERE!
# FEATURE REQUEST
## Installing from git repo
### Pls add this now, kthx

^ The feature request train continues chugging along! Now we'd really like to install from git repos directly. That should be easy right?

---

# Git Dependencies

^ Conceptually these are very similar to folder dependencies. You clone the git repo, check out a committish and then package everything up and continue. They do involve using an external tool for downloading, but otherwise act much like local directory dependencies.

^ You know that build-while-packaging step that npm has that we mentioned earlier? Yeah, well, it doesn't do that for git deps currently. This is a plus because it's MUCH faster because of this, but a minus because some modules just can't be installed without it.  We likely will make it do the build step (if present) in the long run, but it will make those installs MUCH slower.

---

# Wrapping up!

Particularly scary bits we didn't cover:
* gently-rm
* shrinkwrap versus package.json
* shrinkwrap versus bundled deps versus package.json
* … and more!

^ There are a host of areas of additional complexity. There area  whole host of obscure features that hardly anyone knows about that help contribute to the geometric explosion of complexity.

^ I personally hope that as time goes on we can begin to eliminate some of the more obscure / less well thought out features because doing so will substantially simplify npm's design.

---

## Rebecca Turner

### CLI Engineer at
### ![right 25%](npm-logo.png)

### @ReBeccaOrg
### Github, IRC: iarna

^ Oh! And before I leave, be sure to stop by our booth for questions and swag. Everyone gets stickers! Good questions get t-shirts! We've got folks there all day, and the rest of us will be filtering in and out.
