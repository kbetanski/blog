---
title: "Using Hugo modules for themes"
description: "How to use the easiest way to handle theme versioning inside your Hugo website."
date: "2022-03-22"
lastmod: "2022-03-22"
toc: true
tags:
  - jamstack
  - hugo
  - modules
  - themes
links:
  - title: Hugo
    description: 
    website: https://gohugo.io 
  - title: Hugo Theme Stack
    description: Hugo theme I use to create this website. Kudos for @CaiJimmy!
    website: https://github.com/CaiJimmy/hugo-theme-stack
---

Hugo is one of the most popular static site generators. It's a simple tool to
start building a website with ready-to-use themes or your own templates. Let's
focus on using the themes.

## Problem to solve

There are a few ways to handle storing and versioning themes in your Hugo
project. Previously, I used simple vendoring by copying all of the content from
the theme repository to mine, but that's inefficient. It complicates being
up-to-date with new versions of the used theme. Of course, it can be scripted.
But then you have to maintain the script and handle edge cases. For example,
rolling back versions. Also, storing vendored theme versions in the repository
creates big diffs in the git repository in case of updates. It obscures progress
in creating content. Who doesn't like to watch a number of lines going up while
writing content?

My second previous way to handle the theme in the Hugo project was git
submodules. Pros: versioning handled by git. Cons: who understand how does it
work? I worked with git submodules in multiple projects. They're always a
constant headache. Just do not.

## Easy tool

Hugo embraced the usage of the go modules. Go modules is the official way to
handle dependencies inside the go projects. They were introduced back in 2019
and presently are used in most of the go projects. In the case of Hugo, they can
be used as a for:
- static
- content
- layouts
- data
- assets
- i18n
- archetypes

Hugo comes with wrapped `go mod` commands inside. That means you don't have to
install go toolset.

## Solution

Let's use Hugo (go) module as a theme inside the project:

### Initialize the project as a module.

The project also has to be a module itself to use modules. Run command to
initialize the module inside your repository with your URL.
```bash {linenos=false}
# Replace `github.com/kbetanski/blog` with your own repository
hugo mod init github.com/kbetanski/blog
```

The command will create a file `go.mod` inside the project.

```
module github.com/kbetanski/blog

go 1.17
```

### Use theme repository as a module.

Disclaimer: **It works only for a theme that is initialized as a module.**

The module has to be defined inside the configuration file to use it inside the
Hugo project. In my case, it's `config.yaml` file:

```yaml
# Replace path with a theme of your choice
module:
    imports:
        - path: github.com/CaiJimmy/hugo-theme-stack/v3
          disable: false
```

### Download used modules

The last step for using Hugo modules is to download them for usage:
```bash {linenos=false}
hugo mod get
```

The `go.mod` file will be updated with the required dependency.

```
module github.com/kbetanski/blog

go 1.17

require github.com/CaiJimmy/hugo-theme-stack/v3 v3.11.0 // indirect
```

Also, it will create a `go.sum` file. That file contains direct and indirect
dependencies to the project and stores checksums. Each time Hugo downloads the
dependencies, it will validate checksums of what was downloaded against the
stored checksum in that file. In my case file was generated with the following
content:

```
github.com/CaiJimmy/hugo-theme-stack/v3 v3.11.0 h1:HyHdT59BYMdPSjDljIWpJY/DT+NPiZgfqMlJBQwOa1A=
github.com/CaiJimmy/hugo-theme-stack/v3 v3.11.0/go.mod h1:IPmCXiIxlFSLFYS0tOmYP6ySLviyeNVSabyvSuaxD+I=
```

### Updating modules

So, how easy is updating a theme when used as a module? That's how it looks:

```bash {linenos=false}
# All modules
hugo mod get -u

# One module
hugo mod get -u github.com/CaiJimmy/hugo-theme-stack/v3

# One module with the specific version
hugo mod get -u github.com/CaiJimmy/hugo-theme-stack/v3@v3.11.0
```

## Summary

Hugo modules are an easy way to handle dependencies in the Hugo project. As
mentioned before: themes are only one of the possibilities to use them.
Just make sure to use its potential for better content creation!
