---
title: 如何 fix vscode server 不支持低版本 glibc 的问题
---

# {{ $frontmatter.title }}

本文解决问题方案参考 https://feng.moe/posts/202402-fix-vscode-ssh-on-my-server/ 和 https://code.visualstudio.com/docs/remote/faq#_can-i-run-vs-code-server-on-older-linux-distributions 。

本来笔者是想按官网的教程使用 `crosstool-ng` 自己编译一个 `glibc` 。但是由于 `Hygon` 的 CPU 太烂了，等报错出现都要等一个晚上，所以我最终还是上网找了下二进制文件如何配置。

首先要下载 **glibc** 和 **libstdc++** 这两个包，然后我们要用 `rpm2cpio` 和 `cpio` 这两个工具来解压到我们需要的目录。

下载的地址是 https://rpmfind.net/ 。我这里下载的版本依次是

- glibc-common-2.39-56.el10.x86_64.rpm
- libstdc++-14.3.1-2.1.el10.x86_64.rpm
- glibc-2.39-56.el10.x86_64.rpm

接下来，我们就要用下面的命令使用 `rpm2cpio` 和 `cpio` 工具来把这个拷贝到指定目录，完成用 `patchelf` 拷贝到我们的用户态目录，并且指定对应的环境变量实现 `VSCODE` 的兼容。然而我的 login 节点连这都搞不定。

 于是我又去 NJU 镜像下压缩包打算手动构建，结果报错

```
checking whether autoconf works... yes
configure: error:
*** These critical programs are missing or too old: as ld gcc make
*** Check the INSTALL file for required versions.
```

于是哥们手动 `module load` 了 `devtoolset` 并且手动构建了 `make`。

然后终于可以开始编译了，然而我们的 C86 编译的速度还是太慢，虽然解决了，后来我觉得一定有人准备好了懒人包吧，果不其然找到了懒人包 https://github.com/liuliping0315/glibc2.28_for_CentOS7 。

按照这个的 README 做就可以直接 fix vscode server 不支持低版本 glibc 的问题 。