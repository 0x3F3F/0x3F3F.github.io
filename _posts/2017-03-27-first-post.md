---
layout: post
section-type: post
title: First Post
category: tech
tags: [ 'intro' ]
---

This is my first post, just testing it out.  Initial thoughts: Front page looks good, but the centre alignment looks bad on individual blog posts (update: sort of fixed).

The site is hosted on Github and is powered by Jekyll.  To set up, I created a `USER.github.io` repository with a fork of [personal-jekyll-theme](https://github.com/PanosSakkos/personal-jekyll-theme) and aded a `CNAME` file in the root with my www domain.  Other than that, I just needed to then create `CNAME` record pointing to git and `A` records(x2) at my domain registrar.  I'll probably hack about with this theme a bit.

The posts are constructed in markdown, which should be great.  I've put some tests below to see how stuff renders.  In theory, it should all just work, but we'll see how it goes.

Not sure how often I'll post, probably very infrequently and it's likely to be tech or finance related.

#### Markdown Format Tests

Here is an `inline` and a list:

- one
  - sub one
  - sub two
  - sub three
- two
- three

0. one
0. two
0. three

Hmm, don't like the centre justification there(fixed).  Lets try indentation:

    one
    two 
    three
  
page separator

---
  
Hmm, need to think of some more markdown that I might want to use... 

