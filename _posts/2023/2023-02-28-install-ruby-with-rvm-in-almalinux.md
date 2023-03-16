---
layout: post
title:  "AlmaLinux下通过rvm安装ruby-3.2.0报错"
date:   2023-02-28 14:17:35 +0800
categories: Dev
---
执行如下指令进行ruby安装，结果出错:

```
[gem@h113 ~]$ rvm reinstall ruby-3.2.0
ruby-3.2.0 - #removing src/ruby-3.2.0..
ruby-3.2.0 - #removing rubies/ruby-3.2.0..
Searching for binary rubies, this might take some time.
No binary rubies available for: centos/8/x86_64/ruby-3.2.0.
Continuing with compilation. Please read 'rvm help mount' to get more information on binary rubies.
Checking requirements for centos.
Requirements installation successful.
Installing Ruby from source to: /home/gem/.rvm/rubies/ruby-3.2.0, this may take a while depending on your cpu(s)...
ruby-3.2.0 - #downloading ruby-3.2.0, this may take a while depending on your connection...
ruby-3.2.0 - #extracting ruby-3.2.0 to /home/gem/.rvm/src/ruby-3.2.0.....
ruby-3.2.0 - #configuring..................................................................
ruby-3.2.0 - #post-configuration..
ruby-3.2.0 - #compiling..................................................................................................
ruby-3.2.0 - #installing................
Error running '__rvm_make install',
please read /home/gem/.rvm/log/1677551981_ruby-3.2.0/install.log
There has been an error while running make install. Halting the installation.
```

根据提示信息查看报错日志/home/gem/.rvm/log/1677551981_ruby-3.2.0/install.log，有如下信息：
```
<internal:/home/gem/.rvm/src/ruby-3.2.0/lib/rubygems/core_ext/kernel_require.rb>:85:in `require': cannot load such file -- psych (LoadError)
from <internal:/home/gem/.rvm/src/ruby-3.2.0/lib/rubygems/core_ext/kernel_require.rb>:85:in `require'
from /home/gem/.rvm/src/ruby-3.2.0/lib/rubygems.rb:610:in `load_yaml'
from /home/gem/.rvm/src/ruby-3.2.0/lib/rubygems/config_file.rb:346:in `load_file'
```
往前翻翻：
```
make[1]: Entering directory '/home/gem/.rvm/src/ruby-3.2.0'
*** Following extensions are not compiled:
psych:
Could not be configured. It will not be installed.
Check ext/psych/mkmf.log for more details.
```

查看/home/gem/.rvm/src/ruby-3.2.0/ext/psych/mkmf.log

 
```
LD_LIBRARY_PATH=.:../.. "gcc -I../../.ext/include/x86_64-linux -I../.././include -I../.././ext/psych -O3 -fno-fast-math -ggdb3 -Wall -Wextra -Wdeprecated-declarations -Wdiv-by-zero -Wduplicated-cond -Wimplicit-function-declaration -Wimplicit-int -Wmisleading-indentation -Wpointer-arith -Wwrite-strings -Wold-style-definition -Wimplicit-fallthrough=0 -Wmissing-noreturn -Wno-cast-function-type -Wno-constant-logical-operand -Wno-long-long -Wno-missing-field-initializers -Wno-overlength-strings -Wno-packed-bitfield-compat -Wno-parentheses-equality -Wno-self-assign -Wno-tautological-compare -Wno-unused-parameter -Wno-unused-value -Wsuggest-attribute=format -Wsuggest-attribute=noreturn -Wunused-variable -Wundef -fPIC -c conftest.c"
conftest.c:3:10: fatal error: yaml.h: No such file or directory
#include <yaml.h>
^~~~~~~~
compilation terminated.
checked program was:
/* begin */
1: #include "ruby.h"
2:
3: #include <yaml.h>
/* end */

--------------------
```
根据报错信息，分析yaml.h是 libyaml-devel的文件，于是尝试通过如下语句安装对应开发包: 
```
[gem@h113 ~]$ sudo yum install libyaml-devel
Last metadata expiration check: 1:42:30 ago on Tue 28 Feb 2023 12:40:37 PM CST.
No match for argument: libyaml-devel
Error: Unable to find a match: libyaml-devel
```
显示找不到安装包。libyaml-devel是在powertools里，这个repo默认是禁用的，需要通过如下方式安装。
```
yum --enablerepo=powertools install libyaml-devel
```