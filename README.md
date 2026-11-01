# 0xBerk0.github.io

```
 0xBERK  //  RED TEAM OPERATOR // OFFENSIVE DEVELOPER // PT
```

Source for **[0xberk0.github.io](https://0xberk0.github.io)**: writeups, offensive tooling and
Windows internals research. Jekyll + a custom synthwave / CRT theme that matches the GitHub profile banner.

## layout

```
.
├── index.html                 # home: banner, loadout, featured arsenal, ops.log, whoami
├── arsenal.md                 # /arsenal/  → _layouts/arsenal.html
├── writeups.md                # /writeups/ → _layouts/writeups.html (search + filters)
├── whoami.md                  # /whoami/   → _layouts/whoami.html
├── archives/index.html        # old /archives/ URL → redirects to /writeups/
├── _data/
│   ├── profile.yml            # handle, bio, focus, certs + progress, loadout, stats
│   └── arsenal.yml            # tooling repos, grouped (featured: true → homepage)
├── _layouts/                  # default, post, page, writeups, arsenal, whoami
├── _includes/                 # topbar, footer
├── _posts/                    # writeups (markdown)
└── assets/css/site.css        # the whole theme
```

## adding a writeup

```yaml
---
layout: post
title: "HackTheBox — Machine"
date: 2026-09-01
categories: [HTB, Active Directory]
tags: [kerberoast, dcsync]
description: "One-line attack chain: foothold → privesc → DA."
platform: "HackTheBox"
os: "Windows"
difficulty: "Medium"      # easy | medium | hard | insane
---
```

## updating the profile

- certification progress → `_data/profile.yml` → `certs[].progress` (100 = cleared)
- new tool → `_data/arsenal.yml` (set `featured: true` to show it on the homepage)

## local

```sh
bundle install
bundle exec jekyll serve
```

---

Everything here is for education and authorized testing only. struggle. adapt. understand.
