---
layout: post
title:  "Debian Cleanups"
date:   2016-04-12 23:30:30 +0100
categories: debian
---
Today I cleaned up some of the default installation configuration that does
not please me. Here are the actions I've done:

## Remove Bind and Exim
- Remove unneeded packages
{% highlight shell %}
[root@db-sc1 ~]# apt remove bind9 exim4 mutt exim4-base exim4-config exim4-daemon-light
[root@db-sc1 ~]# apt purge bind9 exim4 mutt exim4-base exim4-config exim4-daemon-light
[root@db-sc1 ~]# apt install bind9utils
[root@db-sc1 ~]# rm /etc/cron.daily/exim4-base
{% endhighlight %}

## Install zsh and some nice tools
{% highlight shell %}
[root@db-sc1 ~]# apt install tcpdump
[root@db-sc1 ~]# apt install install zsh git zsh-doc
[claer@db-sc1 ~]$ mkdir .zsh
[claer@db-sc1 ~]$ cd .zsh
[claer@db-sc1 ~]$ git clone git://github.com/zsh-users/zsh-syntax-highlighting.git
{% endhighlight %}

- Create the $HOME/.zshrc script that match my usage
{% highlight shell %}
zstyle ':completion:*' completer _expand _complete _approximate
zstyle :compinstall filename '/home/claer/.zshrc'
autoload -Uz compinit
compinit
autoload -U colors
colors
HISTFILE=~/.histfile
HISTSIZE=4000
SAVEHIST=4000
unsetopt beep
bindkey -e

export PROMPT='[%n@%m %!%(?. %~]. %~] %? )%# '
PATH=$PATH:/usr/games/bin:$HOME/bin
export PATH
setopt INC_APPEND_HISTORY HIST_IGNORE_ALL_DUPS HIST_SAVE_NO_DUPS
alias ll='ls -l'

# source zsh syntax highlight
source "${HOME}/.zsh/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh"
{% endhighlight %}

- Update the running shell for the user
{% highlight shell %}
claer@db-sc1:~$ chsh
Password:
Changing the login shell for claer
Enter the new value, or press ENTER for the default
        Login Shell [/bin/bash]: /usr/bin/zsh
claer@db-sc1:~$
{% endhighlight %}

