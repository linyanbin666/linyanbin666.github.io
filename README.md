# [linyanbin666.github.io](https://linyanbin666.github.io/)

[![GitHubPages](https://github.com/linyanbin666/linyanbin666.github.io/workflows/build/badge.svg)](https://github.com/linyanbin666/linyanbin666.github.io/actions)
[![Generator is Hugo](https://img.shields.io/badge/Generator-Hugo-ff4088?&logo=hugo)](https://github.com/gohugoio/hugo)
[![Theme is MemE](https://img.shields.io/badge/Theme-MemE-2a6df4)](https://github.com/reuixiy/hugo-theme-meme)

The `master` branch contains all source codes for the page. Generated contents are deployed to the [`docs`](https://github.com/linyanbin666/linyanbin666.github.io//tree/docs) branch.

## Get Started

### Prerequisite

Install [**Hugo**](https://github.com/gohugoio/hugo/) first. Note that this repository use the hugo extended version.

### Clone & update submodules (themes)

```
$ git clone --recursive git@github.com:linyanbin666/linyanbin666.github.io.git
```

### Update sub modules for themes

```
$ git submodule update --init --recursive --remote
```

### Write new post

```
$ hugo new posts/article.md
```

### Start server locally

```
$ hugo server -D
```
    
### Build

```
$ hugo
```

### Deploy with GitHub Actions

This repository has been configured for deployment with GitHub Actions. See [Actions](https://github.com/linyanbin666/linyanbin666.github.io/actions).
