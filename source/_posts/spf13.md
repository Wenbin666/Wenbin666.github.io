---
title: spf13环境部署
date: 2021-05-13 21:21:53
tags: vim
categories:
- environment
---

# spf13环境部署
ubuntu下通过命令进行安装

```bash
curl https://j.mp/spf13-vim3 -L > spf13-vim.sh && sh spf13-vim.sh
```
如果命令执行失败，可以通过先把sh文件下载下来，然后执行sh spf13-vim.sh。

# gtags安装
1. 通过命令安装

```bash
sudo apt-get install global
```

2. 通过安装包进行安装

```bash
wget https://ftp.gnu.org/pub/gnu/global/global-6.6.tar.gz
tar -xvf ./global-6.6.tar.gz
./configure
make -j4
make ~/opt
export PREFIX=~/opt
make install
echo export PATH=\$PATH:~/opt/usr/local/bin >> ~/.bashrc
source ~/.bashrc
```
# 适配gtags到vim
```
mkdir ~/.vim/plugin
cp ~/opt/usr/local/share/gtags/*.vim ~/.vim/plugin/
```

# 配置vim 自定义项目（选配）
".vimrc"文件添加快捷键和高亮内容
```bash
abbreviate gt Gtags
"Highlight all search pattern matches•
set hls
"Map F12 to create ctags index in current directory•
map <F12> :!ctags -R <CR><CR>
"A shotcut to execute the grep command•
map mg :!grep <C-R><C-W> . -r <CR>
"change the comment color•
hi Comment ctermfg=6
```

