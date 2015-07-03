## 概要

昨晚基本折腾了一个通宵的capistrano、unicorn和spring，然而还是没能完全实现rails应用的从零部署。不过capistrano确实是令人惊叹的强大，方便的api以及对DevOps友好的语言，一键部署指定环境，能让开发本地直接连接指定环境的rails console等，还有很多特性还没来得及了解，难怪我们这边连nodejs应用都采用capistrano部署。[参考文章](https://ruby-china.org/topics/18616)

## cap文件结构

cap的文件结构分为两部分，一部分是负责部署任务的development文件结构，另一部分是服务器上的文件结构。在执行install指令后会生成这么一个文件结构

```
├── Capfile
├── config
│   ├── deploy
│   │   ├── production.rb
│   │   └── staging.rb
│   └── deploy.rb
└── lib
    └── capistrano
            └── tasks
```

其中比较重要的是deploy.rb及deploy文件夹下的各个环境配置，自动生成的配置和注释很容易读懂，基本照着改就能完成最简单的ruby应用部署了。

服务器的文件结构如下

```
├── current -> /home/ares/apps/appname/releases/20140325071623
├── releases
│   ├── 20140325065734
│   ├── 20140325071310
│   ├── 20140325071623
│   └── 20140325074922
├── repo
│   ├── branches
│   ├── config
│   ├── description
│   ├── FETCH_HEAD
│   ├── HEAD
│   ├── hooks
│   ├── info
│   ├── objects
│   ├── packed-refs
│   └── refs
├── revisions.log
└── shared
    ├── bin
    ├── bundle
    ├── config
    ├── log
    ├── public
    ├── tmp
    └── vendor
```

这和当年大爷为ac做的部署系统异曲同工，就差个web ui了，这也是一目了然的结构，非常方便项目无缝地切换，部署和回滚。不过这在不支持soft link的cifs下完全没法用。

## cap flow

- Deploy flow

```
deploy:starting    - start a deployment, make sure everything is ready
deploy:started     - started hook (for custom tasks)
deploy:updating    - update server(s) with a new release
deploy:updated     - updated hook
deploy:publishing  - publish the new release
deploy:published   - published hook
deploy:finishing   - finish the deployment, clean up everything
deploy:finished    - finished hook
```

- Rollback flow

```
deploy:starting
deploy:started
deploy:reverting           - revert server(s) to previous release
deploy:reverted            - reverted hook
deploy:publishing
deploy:published
deploy:finishing_rollback  - finish the rollback, clean up everything
deploy:finished
```

当配置插件后，插件自身会在这flow里添加很多自己的流程，cap提供的before或after方法来插入任务也是简单易懂。这边顺便贴个官网的例子（和我们的flow其实基本以模样）

- example capfile

```
# Capfile
require 'capistrano/setup'
require 'capistrano/deploy'
require 'capistrano/bundler'
require 'capistrano/rails/migrations'
require 'capistrano/rails/assets'
```
- example deploy flow

```
deploy
  deploy:starting
    [before]
      deploy:ensure_stage
      deploy:set_shared_assets
    deploy:check
  deploy:started
  deploy:updating
    git:create_release
    deploy:symlink:shared
  deploy:updated
    [before]
      deploy:bundle
    [after]
      deploy:migrate
      deploy:compile_assets
      deploy:normalize_assets
  deploy:publishing
    deploy:symlink:release
  deploy:published
  deploy:finishing
    deploy:cleanup
  deploy:finished
    deploy:log_revision
```
- example rollback flow

```
deploy
  deploy:starting
    [before]
      deploy:ensure_stage
      deploy:set_shared_assets
    deploy:check
  deploy:started
  deploy:reverting
    deploy:revert_release
  deploy:reverted
    [after]
      deploy:rollback_assets
  deploy:publishing
    deploy:symlink:release
  deploy:published
  deploy:finishing_rollback
    deploy:cleanup_rollback
  deploy:finished
    deploy:log_revision
```

可以在cap的时候带上```--trace```执行一遍打印流程,对于devops来说主要还是关注这flow中的一些hook点。

## 坑点和心得
cap本身配置挺简单，并且**环境单独的配置覆盖公共的配置**但插件配置真是有些蛋疼（主要还是不熟）。最开始我是直接复制我们的production文件来配local的stage，结果碰到了一些坑，主要问题还是在于我对于cap/rvm这个项目以及cap的roles系统的不熟悉。有个折腾了我一晚上的问题，cap/unicorn的事件怎么都不触发，第二天一问潘神，一说roles我就恍然大悟。cap可以用```on roles(:role)```来设定执行这个任务的角色，我修改了local.rb的roles列表，却忘了匹配上deploy.rb执行unicorn:duplicate的role，酿成了接近2个小时的悲剧。所以有时候在半夜干活点天赋也不太合适，想请教问题都找不到人。（渣英文，找不来老外）

最后还有些疑问，在项目里闪总写了一段有一些关于rails assets的task，after assets:precomile事件，里面的方法很奇怪，capture方法是什么鬼，upload!又是什么鬼。。google了一晚上找不到头绪，（capture撞到验证码了，反而完全插不到资料）对于前端文件编译合并的部分我也确实是完全不熟，只能等有空再来坑了。