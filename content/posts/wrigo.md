+++
author = ["Kalin Daskalov"]
title = "wrigo"
date = "2023-09-04"
description = "helper tool for new hugo posts"
tags = ["go", "code", "bash",]
+++

When I first went into DevOps professionally, whenever an automation oppurtunity arose, my first instinct was to use bash for whatever script/tool needed to be built. That's what I've been using for my personal needs and what I'm most familiar with.

I want to learn Go through use though and I'm actively trying to circumvent my instinct for bash, even for simple things. This post has metadata in the beginning you see as formatted title, date, description, tags, etc. They're in the beginning of each <post>.md file like this for example:

```
+++
author = "System Scribe"
title = "wrigo"
date = "2023-09-04"
description = "helper tool for new hugo posts"
tags = ["go", "code", "bash",]
+++
```

I had a few thoughts regargind this:

1. Copy the last entry from each last post and edit it.
2. Create a template I fill each time.
3. Create a bash script to generate a new post file with filled metadata

I was set, catually on `number 3`, because writing in bash is always a fun experience but I reminded myself that the thing I want more was to exit a bit this comfort zone and try to use go for the job. So we got to:

4. Create Go program to generate a new post file with filled metadata with persistance of some fields

I knew it will be more complicated, more searching, more lines of code, etc. It actually took only a Sunday afternoon but was a good excercise. I've written two web scrapers about two months ago and felt rusty but things kept coming back. I reminded myself that maybe by consistently writing and progressively complicating such small innocent projects, I'll learn the language better than going after a mastodon project and rewriting it everything I consider a new way of doing it better.

So it's not much, definitely not a portfolio project for the future but a good start for this blog.

I'm following advice on how to structure my files and code by a friend but I've only processed only about 5% of what he's told me for now.

You can check the code on my [wrigo](https://gitlab.com/systemscribe/wrigo) repo. Including the readme as well:

# Wrigo

Wrigo (as in "Writer for Hugo") is a simple tool to generate the metadata for your Hugo posts in the terminal.

## Features

1. Generates the following metadata:
    - Author
    - Title
    - Description
    - Current Date
    - Tags
2. Persists author and previous tag data in `~/.config/wrigo/config.yaml`. This way in the next prompt, you don't need to set author again and see last used tags.
3. Generates `<lower-case-of-your-title>.md` file in the current directory.
4. Makes starting a new post less of a hassle than using templates.

## Get started

```bash
# Clone this repo
git clone gitlab.com/systemscribe/wrigo
cd wrigo

# Build a binary in your $PATH
go build -o ~/.local/bin/wrigo cmd/main.go

# Generate a new markdown file with metadata
wrigo
```

