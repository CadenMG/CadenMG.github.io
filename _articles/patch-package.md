---
id: 5
title: "Patching Node Dependencies"
subtitle: ""
date: "2024.08.27"
tags: "node, npm, patch-package"
---

## Introduction
I had a problem recently which was at the intersection of multiple newer pieces of technology. Because of this, there weren't many great libraries to pull from to make to my life easier - I'd have to just manually implement a lot of it... But there were some solutions.
And these few solutions were often broken in some way.

I could clone the repo, make my change, and publish to npm.
Or even push a fix PR and pray the 1 maintainer reviews it.

## Patches
Or you don't have to do any of that.

[patch-package](https://www.npmjs.com/package/patch-package) lets you instantly make fixes to your dependencies. It's as simple as:

```
# edit your dependency
edit node_modules/@package/file.js

# run patch-package to create a patches/ directory and .patch file
npx patch-package @package

# patch can be commited and shared with team
git add patches/
```

Now you can apply those patches with `npx patch-package` (no arguments).

If you don't want to do this every time you install, you can add a `postinstall` script like:
```
"scripts": {
+  "postinstall": "patch-package"
 }
```

It even provides you with a convenient way of creating issues against the source repo based on your changes:
```
npx patch-package @package --create-issue
```
