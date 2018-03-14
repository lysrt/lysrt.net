---
title: "Hugo Workflow"
date: 2018-02-25T23:41:52+01:00
draft: true
---

Hugo is a static website generator. It is the tool I'm using to create the pages on my site lysrt.net.

For those who don't know how a static site generator works, basically it takes layout and content files, and "compiles" them into browser readable HTML. In the case of Hugo, templates are Go HTML templates, and content files are Markdown files.

The output of the file generator is a folder called `public`, with HTML and static files, ready to be browsed.

For an example of what it looks like, please check out the website of Hugo, or my [website github repo](https://github.com/lysrt/lysrt.net).

It took me a bit of thinking to properly organize the source and the output of my website. Let's see what options we have out there, and I will detail my current workflow.


