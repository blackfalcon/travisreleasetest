[![Build Status](https://travis-ci.org/blackfalcon/travisreleasetest.svg?branch=master)](https://travis-ci.org/blackfalcon/travisreleasetest)
![npm (scoped)](https://img.shields.io/npm/v/@blackfalcon/travisreleasetest.svg?color=orange)
[![Greenkeeper badge](https://badges.greenkeeper.io/blackfalcon/travisreleasetest.svg)](https://greenkeeper.io/)

# Semantic Release with Travis CI

> For the sake of demonstration, this repository is a simple little thing that will take a Scss file and output a CSS file in a `dist/` dir.

Like me, you may have built a project and wanted to release it to the world! YAY! That's awesome!

But also like me, when you get to the RELEASE part, that's where the sadness sets in. Yes, it is as easy as pie to take your code and do a local build, manually update the version number and publish to npm, but that process sucks and it doesn't scale.

It's at this point where a few things may come into play.

1. Automated build process
1. Automated version management
1. Automated CHANGELOG.md creation/updates

It's at this point that you feel that not only do you have bad-ass code, but you also have a bad-ass deployment process.

## Travis CI

Now you could use whatever CI/CD platform you want, but for the sake of demonstration I am going with [Travis CI](https://travis-ci.org).

Setting up a Travis CI account is easy. If you have a Github account, you have a Travis account. Done.

First things first. To get this demo working you will need to set up a few things.

### .travis.yml file

In your repo you will need to create a `.travis.yml` file. This is needed so that when you go to [travis-ci.org](https://travis-ci.org) you can enable your repository.

At the very least you should have something like the following:

```yaml
language: node_js

node_js:
  - 8
```

Commit that to your repo and then head over to [travis-ci.org](https://travis-ci.org) and using their discovery tools, find your repo and enable it.

### Environment variables

Now that you are in Travis CI's tools with your repo enabled, look for the `MORE OPTIONS` toggle and select `settings`.

In here you will want to configure three environmental variables;

1. GITHUB_TOKEN   // go to your Github account and generate the token
1. NPM_EMAIL      // this is the email address per your npmjs.org account
1. NPM_TOKEN      // in your account settings, generate a token

These will be needed for automated deployment to npmjs.org and allow for updates to be committed back to your repo.

In the next steps we will come back to the `.travis.yml` file, but for now, you are done configuring within the site.

## package.json

90% of the magic to make this work will happen in the `package.json` file once configured. To get things rolling, you will need to install some packages.

```
$ npm i semantic-release -D
$ npm i @semantic-release/changelog -D
$ npm i @semantic-release/git -D
$ npm i @semantic-release/npm -D
$ npm i @commitlint/cli -D
$ npm i @commitlint/config-conventional -D
$ npm i husky -D
```

### Husky

Husky is a cool tool for managing pre-commit hooks. In this demo I am using Husky to run the test PRIOR to the commit and lint the message for the correct format. The main thing I am looking for are the Angular style commits. This is needed so that Semantic Release can evaluate the commit message and create the correct version update.

```json
"husky": {
  "hooks": {
    "pre-commit": "npm test",
    "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
  }
},
"commitlint": {
  "extends": [
    "@commitlint/config-conventional"
  ]
}
```

Once this is completed, while this will not automate releases yet, it will lint your messages.

### Commitlint

The following examples illustrate how different commit messages will evaluate to different release versions.

#### MAJOR

For a MAJOR release, you MUST follow this template. The use of `perf:` and `BREAKING CHANGE:` are needed in order to push a major release.

```
perf(pencil): remove graphiteWidth option

BREAKING CHANGE: The graphiteWidth option has been removed.
The default graphite width of 10mm is always used for performance reasons.
```

#### MINOR
```
feat(pencil): add 'graphiteWidth' option
```

#### PATCH
```
fix(pencil): stop graphite breaking when too much pressure applied
```

#### Other commit types

* `docs:` Documentation only changes
* `style:` Changes that do not affect the meaning of the code (white-space, formatting, missing semi-colons, etc)
* `refactor:` A code change that neither fixes a bug nor adds a feature
* `perf:` A code change that improves performance
* `test:` Adding missing or correcting existing tests
* `chore:` Changes to the build process or auxiliary tools and libraries such as documentation generation

### Release

**semantic-release**’s options, mode and plugins can be set via:

* A `.releaserc` file, written in `YAML` or `JSON`, with optional extensions: `.yaml/.yml/.json/.js`
* A `release.config.js` file that exports an object
* A release key in the project's `package.json` file

[read more](https://github.com/semantic-release/semantic-release/blob/HEAD/docs/usage/configuration.md#configuration-file)

For the sake of this demo, the configs are in the `package.json` file.

The example pretty much speaks for itself. Mainly, you want to remember to only deploy on `master` or Travis will double deploy your project. Once on commit from you and then when semantic-release commits the tag.

In the `plugins` is where more magic happens, specifically the generation of the `CHANGELOG` file and taking the information from the preparation to publish life-cycle, the version in the `package.json` will get updated and both are committed back to the repo.

```json
"release": {
  "branch": "master",
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    [ "@semantic-release/changelog", {
      "changelogFile": "./CHANGELOG.md",
      "changelogTitle": "# Semantic Release Automated Changelog"
    }],
    "@semantic-release/npm",
    [ "@semantic-release/git", {
      "assets": [ "./CHANGELOG.md", "package.json" ]}
    ],
    "@semantic-release/github"
  ]
}
```

## Travis CI

See, I said that we would get back to this. Now that we have configured our semantic-release in the `package.json` file, the next step is to update the `.travis.yml` file with the correct sequence of commands.

```yaml
language: node_js

node_js:
  - 8

branches:
  only:
    - master

notifactions:
  email: false

before_script:
  - npm run sass

after_success:
  - npx semantic-release
```

What I didn't understand initially was that semantic-release does all the work that the `deploy` config does with Travis. So there was a lot of confusion between how semantic-release works and how Travis works. And my build was trying to deploy twice on Travis?

So, do not use the `deploy:` config as illustrated in many of the Travis docs.

Things to remember are, `before_script:` will be where you perform and build options to be included in the npm package you are distributing.

Once that has completed, Travis will run `npm test`. So, just be aware of that.

After your pre-build tasks have been completed, and your test(s) pass, then in the `after_success:` config, run `npx semantic-release`. This is where semantic-release will fully take over, evaluate your commit, apply the version number, generate the package, release the package, commit the updates `package.json` and the `CHANGELOG.md` file back to the repo, apply the tag, and generate the project's binaries.

That is an amazing mouthful of events and it's all out of your hands!

That's what makes this so cool. The idea that all of this happens without personal intervention and by establishing a commit message standard, you are assured that others that commit to your project will always comply with process.

## Greenkeeper

Now Greenkeeper is a pretty awesome service and NOT NEEDED for semantic-release to work. I simply wanted to test out this service and what better repo to use it on then this. Why not?

If you are interested in Greenkeeper, simply click in the badge in this repo and it will walk you through the steps to set up Greenkeeper on your repo.
