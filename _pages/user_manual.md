---
layout: single
classes: wide
title: User manual
permalink: /user_manual/
toc: true
toc_icon: "cog"
---

## Troubleshooting

* How to stop all docker containers

```bash
docker stop $(docker ps -a -q) && docker rm $(docker ps -a -q)
```
