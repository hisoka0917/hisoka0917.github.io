---
layout: post
title:  npm在升级之后报错
date:   2017-12-29 11:06:00 +0800
categories: nodejs
tags: [npm, nodejs, osx, homebrew]
---

今天在执行npm命令的时候报错，如下所示：

```
$ npm -v
module.js:557
    throw err;
    ^

Error: Cannot find module '../lib/utils/unsupported.js'
    at Function.Module._resolveFilename (module.js:555:15)
    at Function.Module._load (module.js:482:25)
    at Module.require (module.js:604:17)
    at require (internal/module.js:11:18)
    at /usr/local/lib/node_modules/npm/bin/npm-cli.js:19:21
    at Object.<anonymous> (/usr/local/lib/node_modules/npm/bin/npm-cli.js:92:3)
    at Module._compile (module.js:660:30)
    at Object.Module._extensions..js (module.js:671:10)
    at Module.load (module.js:573:32)
    at tryModuleLoad (module.js:513:12)
```

搜了一下发现是因为之前手贱执行了一下`brew upgrade`，结果就这样了。

解决方法是`rm -rf /usr/local/lib/node_modules/` 然后重新安装node&npm。

