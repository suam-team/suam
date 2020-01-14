---
title: "Create a Blog Post with Hugo"
date: 2020-01-14T01:15:59+07:00
author: "P"
draft: false
description: "A short introduction on how to create a blog post with Hugo"
---

In this post, I will present you a short introduction on how to create a blog post, from the the installation of [Hugo](https://gohugo.io/) to publishing your article.

## SUAM Project

For the SUAM project, there are two main repositories responsible for content publishing, [suam-team/raw](https://github.com/suam-team/raw) for static content and Hugo config, and [suam-team/blog](https://github.com/suam-team/blog) which contains `public/` directory for HTML. We have `deploy.sh` for executing and pushing content from a markdown file to a HTML file.

## Let's Play!

1. Install the latest version of Hugo. The most painless way is to use a binary from [Hugo Releases](https://github.com/gohugoio/hugo/releases) and make it available in `PATH`.
2. Clone the [suam-team/raw](https://github.com/suam-team/raw) with all submodules to your workspace. The recommended command to fetch all submodules in parallel is:

```
$ git clone --recurse-submodules -j8 git@github.com:suam-team/raw.git suam
```

3. Create a post with the command below. The post will be available on `content/posts/my-post.md`.

```
$ hugo new posts/my-post.md
```

4. The current theme required us to add some post metadata in the post header. The parameters below are **required** but you can add more, see this [sample post](https://raw.githubusercontent.com/gohugoio/hugoBasicExample/master/content/post/markdown-syntax.md) for reference.

```markdown
---
title: "Create a Blog Post with Hugo"
date: 2020-01-14T01:15:59+07:00
author: "P"
draft: false
description: "A short introduction on how to create a blog post with Hugo"
---
```

5. You can see a live draft version on your machine with `hugo server -D`. Hugo will generate HTML pages from our markdown files and make it available on `http://localhost:1313/blog/`
6. Publishing your content must be done in the following steps:
   1. Run `./deploy.sh <commit-msg>` to automatically generate HTML pages to `public/`, add and commit changes, and push the changes to [suam-team/blog](https://github.com/suam-team/blog).
   2. Add and commit changes on your current workspace and push it to [suam-team/raw](https://github.com/suam-team/raw)

## All Done!

Congratulation! You've just learned on how to contribute with our SUAM blog!