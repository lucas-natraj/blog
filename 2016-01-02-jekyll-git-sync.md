---
layout: post
category : blog
author: Chase Jenkins & Lucas Natraj
tags: [blog, jekyll, github]
title: Static Site Generation with GitHub & Jekyll
---

## Introduction
The sticky-notes blog is generated using Jekyll, from content that is periodically synchronized from github.  
This post describes some aspects on how the different parts of sticky-notes is put together using jekyll, nodejs, nginx and docker.

## Jekyll

[Jekyll](http://jekyllrb.com/) is a 'blog-aware, static-site generator' that is used to power many blogging platforms including [github pages](https://pages.github.com/).

The basic skeleton of a jekyll site usually looks something like this:

```text
.   <--------------------------------------- [base folder]
├── _config.yml
├── _drafts
|   ├── begin-with-the-crazy-ideas.textile
|   └── on-simplicity-in-technology.markdown
├── _includes
|   ├── footer.html
|   └── header.html
├── _layouts
|   ├── default.html
|   └── post.html
├── _posts    <----------------------------- [posts folder]
|   ├── 2015-10-29-hacky-sack-for-programmers.md
|   └── 2016-01-01-pen-tricks.md
├── assets    <----------------------------- [assets to reference in posts]
|   └── hacky-sack-blue.png
├── _data
|   └── members.yml
├── _site     <----------------------------- [generated site]
├── .jekyll-metadata
└── index.html
```

The key folders here are
 
* `_posts` - The 'dynamic content', so to speak. The naming convention of these files is important, and must follow the format `YEAR-MONTH-DAY-title.MARKUP`. Each file content must also contain [YAML Front Matter](http://jekyllrb.com/docs/frontmatter/).
* `assets` - Contains any supporting images / pdfs that are to be referenced in posts.
* `_site` - This is where the generated site will be placed (by default) once Jekyll is done transforming it. 

Running Jekyll on that base folder will generate the `_site` and its contents (which should then be statically served out with a web server).
```bash
$ jekyll build
# => The current folder will be generated into ./_site
```

It can also be configured to continually watch the base folder for file changes to know when to regerate the `_site`.
```bash
$ jekyll build --watch
# => The current folder will be generated into ./_site,
#    watched for changes, and regenerated automatically.
``` 

Jekyll also comes with a built-in (development) server that can serve the generated site
```bash
$ jekyll serve
# => A development server will run at http://localhost:4000/
#    Auto-regeneration: enabled. Use `--no-watch` to disable.
```

## Deployment

**`Update (2016.03.02): The general structure has changed a little. Details below.`**

In order to make creating posts as simple as possible, we have created a simple synchronization application that will fetch 'posts' (and assets) from a github repository and regerate the jekyll site. See diagram below. 

![Diagram]({{site.baseurl}}/assets/2016-01-02-jekyll-git-sync-design.png)

As shown, there are 2 docker containers running on the host. One containing the Jekyll server and another containing the git synchronization application.
There is a shared volume '/site' between both containers that will contain the final jekyll site. And there is also a folder (containing rsa keys) on the host that is mounted as a volume on the git-sync container to provide any credentials (usually for a service account) that are needed to access the git repo. 

Initializing the git-sync application can be triggered via a REST call with the desired github repo & branch information.

`POST git-sync-url:port/init`

```json
{
    "siteRepo": "Schlumberger/sticky-notes",
    "siteRepoCert": "-----BEGIN RSA PRIVATE KEY-----\n ...",
    "siteBranch": "master",
    "siteOffset": "projects/site",
    "contentRepo": "Schlumberger/sticky-notes-content",
    "contentRepoCert": "-----BEGIN RSA PRIVATE KEY-----\n ...",
    "contentBranch": "master",
    "contentOffset": ".",
    "knownHosts": "|1|VSNRJvcG1uGQK4IoyM...",
    "url": "http://server-name:8080"
}
```

Note:
The 'url' property is used when generating the feed (http://server:port/feed.xml) to ensure that references from the feed item to the correct server are correct. 

Once this has been done, the git-sync application will, on a 5 minute interval, query both the 'site' repo (for jekyll-related files, config, themes, ...) as well as the the 'content' repo (for _posts & assets) and sync down any updates.

Once the repositories have been pulled, the site & content folders are then merged into the `/site` folder (shared volume), thereby creating the jekyll site.

Jekyll (running in the other container) will detect the new changes to this folder (shared volume) and then regenerate the site (`_site`) and begin serving the new content.

An immediate sync & regenerate can also be triggered 

`POST git-sync-url:port/update`


## Writing Posts

With this setup writing a new, or editing an existing post, is as straightforward as commiting a new change to the 'content' git repository (specifically within the `_posts` & `assets` folders). These changes will then be picked up the next time the git-sync executes.


## Update (2016.03.02)

Sticky-notes now uses a slightly simplified structure than the one descrbied above.  
Instead both docker containers have now been combined into 1 with the following characteristics:
- Jekyll is no longer 'watching' the target folder for changes (to determine when to regenerate the site).
- The node server still synchronizes content with Git, but will now, upon completion of that task, invoke jekyll to regenerate the site 
- Nginx is used to serve the static content (rather that the built-in jekyll development web server) 
- Since everything is in 1 container there is no need for making the '/site' a shared volume


## References

- [Markdown syntax](http://daringfireball.net/projects/markdown/syntax)
- [Writing & formatting](https://help.github.com/categories/writing-on-github/)
- [Creating and highlighting code blocks](https://help.github.com/articles/creating-and-highlighting-code-blocks/)
