---
layout: post
title: 操作系统安装工具汇总
category: 工具
tags: [brew, apt-get, dpkg, yum]
keywords: brew, apt-get, dpkg, yum
---

## brew

## apt-get
### 基本命令
安装，会自动安装依赖包
```
sudo apt-get --fix-broken install ./xxx.deb
```

## dpkg
### 基本命令
安装，不会自动安装依赖包
```
sudo dpkg -i xxx.deb  
```

查看
```
dpkg -l | grep xxx
```

卸载
```
sudo dpkg -r xxx
```

帮助
```
dpkg --help
```


## yum
### 命令选项:
```
  -h, --help            show this help message and exit
  -t, --tolerant        be tolerant of errors
  -C, --cacheonly       run entirely from system cache, don't update cache
  -c [config file], --config=[config file]
                        config file location
  -R [minutes], --randomwait=[minutes]
                        maximum command wait time
  -d [debug level], --debuglevel=[debug level]
                        debugging output level
  --showduplicates      show duplicates, in repos, in list/search commands
  -e [error level], --errorlevel=[error level]
                        error output level
  --rpmverbosity=[debug level name]
                        debugging output level for rpm
  -q, --quiet           quiet operation
  -v, --verbose         verbose operation
  -y, --assumeyes       answer yes for all questions
  --assumeno            answer no for all questions
  --version             show Yum version and exit
  --installroot=[path]  set install root
  --enablerepo=[repo]   enable one or more repositories (wildcards allowed)
  --disablerepo=[repo]  disable one or more repositories (wildcards allowed)
  -x [package], --exclude=[package]
                        exclude package(s) by name or glob
  --disableexcludes=[repo]
                        disable exclude from main, for a repo or for
                        everything
  --disableincludes=[repo]
                        disable includepkgs for a repo or for everything
  --obsoletes           enable obsoletes processing during updates
  --noplugins           disable Yum plugins
  --nogpgcheck          disable gpg signature checking
  --disableplugin=[plugin]
                        disable plugins by name
  --enableplugin=[plugin]
                        enable plugins by name
  --skip-broken         skip packages with depsolving problems
  --color=COLOR         control whether color is used
  --releasever=RELEASEVER
                        set value of $releasever in yum config and repo files
  --downloadonly        don't update, just download
  --downloaddir=DLDIR   specifies an alternate directory to store packages
  --setopt=SETOPTS      set arbitrary config and repo options
  --bugfix              Include bugfix relevant packages, in updates
  --security            Include security relevant packages, in updates
  --advisory=ADVS, --advisories=ADVS
                        Include packages needed to fix the given advisory, in
                        updates
  --bzs=BZS             Include packages needed to fix the given BZ, in
                        updates
  --cves=CVES           Include packages needed to fix the given CVE, in
                        updates
  --sec-severity=SEVS, --secseverity=SEVS
                        Include security relevant packages matching the
                        severity, in updates
```