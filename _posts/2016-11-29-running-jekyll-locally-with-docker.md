---
layout: post
title:  "Running Jekyll locally with Docker"
desc: "Running Jekyll locally with Docker"
keywords: "docker,jekyll"
date: 2016-10-31
categories: [Docker]
tags: [docker]
icon: fa-code
---

I use Jekyll here and there and sometime forget the whole process of getting started...
So, I just wanted to jot it down for next time.

```bash
$ cd jekyll_project_folder
$ docker run -d --name jekyll -v $(pwd):/srv/jekyll -p 4000:4000 jekyll/jekyll:pages jekyll serve --watch --incremental && docker logs -f jekyll
```

or use Docker Compose

```bash
$ cd jekyll_project_folder
# Create a docker-compose.yml file
$ joe docker-compose.yml
# Insert the content below
jekyll:
    image: jekyll/jekyll:pages
    command: jekyll serve --watch --incremental
    ports:
        - 4000:4000
    volumes:
        - .:/srv/jekyll
# Start it
$ docker-compose up && docker-compose logs -f
```

For more information, check out [https://kristofclaes.github.io/2016/06/19/running-jekyll-locally-with-docker/](https://kristofclaes.github.io/2016/06/19/running-jekyll-locally-with-docker/)
