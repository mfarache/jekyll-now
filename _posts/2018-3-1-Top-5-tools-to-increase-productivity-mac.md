---
layout: post
title: Useful bash commands tools to increase your productivity as a developer using Mac
tags: [ Bash, MacOsx, fzz, pecos, oh-my-zsh  ]
---

A few tips on awesome bash tools you can install to improve your productivity as a developer.

# Why bother?

As developers we spend a lot of time executing bash commands in our terminal. We are not fully inmersed in our IDE, often we swith in to 'black scree mode'. We can type bash commands manually, for sure..but we often make silly mistakes that need to be amended as we are not professional typewriters. 

Other approach is using our cheat sheet full of commands that we have handy, but we need to search for the command, copy and paste and some valuable seconds are lost in those operations.

Lastly but not least used, is asking google. How many times have I search for 'how to search big files recursively greater than 10Mb'. I would be embarrased to share that ... 

I have not make any stat about my personal time spent on the terminal, but if only I could save some keystrokes every time I have to type a command, that would give me a tangible benefit. My guesstimates are that in an hour I can type a hell of commands, mix of find, greps, cat, vi, git commands, change directories,etc. Let's assume 2 commands per minute ( and I'm staying in the conservative side), that is 120 commands/hour. So close tp 1000 commands a day. If I save 2 seconds per command that means nearly 30 minutes more than I can spend in something else.

Examples that occurs to me are

+ Trying to remember the syntax of a complex bash command involving pipes and arguments
+ Execute a command after searching for it
+ Executing git commands
+ Changing directories paths
+ Repetitive actions that could be combined in a single one

I will share 3 tools I installed recently and make my day to day work much more productive

# 1. oh-my-zsh

Oh My Zsh is an open source, community-driven framework for managing your zsh configuration.
It's made of tons of plugins and themes that enable you to customize your terminal experience
Installation on [Oh-My-Zsh][5] so follow the steps in their github page.

## Plugins

Did I say tons of plugins? maybe I should challenge you to to find a plugin that is not there

Most of the plugins allow you to provide bash completion. 
You can figure out yourself the usual suspects, git, docker, etc.

The configuration lies in a file under $HOME/.zshrc. Find the plugins section and the plugins you need to work with. 
The full list of plugins can be found here: [Oh-My-Zsh Plugins][4]

## Themes

oh-my-zsh has a huge amount of themes that you can configure to provide a user-friendly experience for those who spent most of the time with the terminal.

I chosed this just because when you work with git is very useful to know extra information.
Questions like : which branch am I ? do I still have pending files to commit? was I in the middle of a merge operation? 

 ![_config.yml]({{ site.baseurl }}/images/TOOLS_THEME.png)

The full list of themes displaying its look and feel can be found here: [Oh-My-Zsh Themes][3]
If you want to select a specific theme you just need to edit the variable ZSH_THEME within $HOME/.zshrc

## Auto suggestions

Autocompletion does not need to be necessarily narrowed down to specific tools. What if the system can gives you suggestions as you type (Google alike). 

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/zsh-autosuggestions
source ~/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh
```

From now on, the plugin will suggest you commands. In case more than one command is available you can swith among them just pressing the arrow keys.


# 2. peco

It's a interactive filtering tool. 

You can pipe the output of any command into pecos and it will display a interactive search filter that will allow to select the entry that best match your criteria.

There are several filters available (IgnoreCase, CaseSensitive, SmartCase, Regexp and Fuzzy) that you can toggle once you are in the pecos search screen.

The tool allows selection of single match, multiple matchs selection and it flexible in the sense that  multiple configuration parameters like layout, colors, behaviour, filter type are available.

Simple usages would be 

```bash
#browse history
history | peco
#inspect processes
ps -aux | peco
#find Python files
find . -name "*.py" -print | peco
```

Defaults work like a charm. I just changed the layout to bottom up as I prefer to look for the filter in the bottom area of my screen

```bash
find . -name "*.py" -print | peco --layout=bottom-up
```

Now that you know about this tool, we can even get a bit further if we automate some commands using alias
I added these aliases to my .bash_rc.profile 

```bash
alias pgit='history | grep git | peco'
alias pcat='ls | peco | xargs cat'
alias pvi='ls | peco | xargs vi'
```

And if you want to get even more benefit give it a go to these shortcuts adding to .bash_rc.profile
These are just sample commands that I need on a daily basis, so feel free to add much more.

```bash

 pfvi () {
 	pfind $1 vi
 }
 
 pfecho () {
 	pfind $1 echo
 }
 
 pfind () {
 	find . -name $1 -print | peco | xargs $2
 }
 
 pgvi () {
 	pgrep $1 $2 vi
 }
 
 pgecho () {
 	pgrep $1 $2 echo
 } 
 
 pgrep () {
	 find . -name $1 -exec grep -l $2 {} \;
 }
```

# 3. fzz

Before I discovered pecos I usedI used to type the Search sequence (Ctrl + R) in order to search for past commands in my history file. Although it works like a charm, it relies on your memory so if you do not remember a specific text bit of command, sometimes you can spend more time than you wanted searching than typing again. 

I think [fzf - command line fuzzy finder][7] does a singe job and very well.
It falls in the category of command line finders with several search patterns like prefix-exact-match,
suffix-exact-match, exact-match, inverse-exact-match and inverse-suffix-exact-match.

```bash
find . -name "*.py" -print | fzf
```

People have experimented using fzf. For example this guy created a tool to keep bookmarks https://dmitryfrank.com/articles/shell_shortcuts.

As we saw with peco, options are infinite once you combine it with alias or functions. A full catalog of cool things can be seen at [fzf - functions examples][9], so give it a go

# 4. The silver search

How many times do you search for specific way of doing things on your favourite language. There is many boilerplate code that you have done in the past and you wish you had handy to reuse some lines, or have a look some trick you did with a library, etc. 

Usually searching involves traversing recursively directories and comparing against some regexp patterns.
grep and egrep are very powefull shell commands, but we cannot tag them as fast. 

In order to search blaze-fast we need a couple of add-ons in our tool set. 
I was testing two options ack and ag([The silver searcher][6]) Both are oriented to search across code files ignoring .git and .svn files.

Eventually after seeing the results and speed, clearly the winner is  ag, way faster and with a lot of options.

```bash
brew install the_silver_search
```
For example lately I've been messing with Python, so is a new language for me. Once I got familiar with it, still I need 
to get back and search simple things like how to iterate over dictionary keys with an index. 

 ![_config.yml]({{ site.baseurl }}/images/TOOLS_AG.png)

Some interesting options like search within zip files or binaries, using regexp expressions make the tool a made match in heaven compared with the verbosity of the grep command sintax and mainly its speed.

```bash
time ag FlinkOutput -G java
0.06s user 0.10s system 118% cpu 0.132 total

time grep -r --include "*.java" FlinkOutput . 
0.26s user 0.21s system 96% cpu 0.488 total

time find . -name "*.java" -exec grep -i 'FlinkOutput' {} \;  
0.65s user 0.63s system 81% cpu 1.567 total

```
The times says it all. 100% faster than simple grep and 600% faster than find & grep combination
With ag results are returned highlighted so you can spot it easily. 
As I'm getting paranoid with saving keystrokes, the command could be trimmed to get exactly one line , then piped into a pbcopy so then u can paste in your IDE. More seconds saved!

# Useful links

+ [Peco][1]
+ [Oh-My-Zsh Autosuggestions][2]
+ [Oh-My-Zsh Themes][3]
+ [Oh-My-Zsh Plugins][4]
+ [Oh-My-Zsh][5]
+ [The silver searcher][6]
+ [fzf - command line fuzzy finder][7]
+ [ag sample usage][8]
+ [fzf - functions examples][9]


[1]: https://github.com/peco/peco
[2]: https://github.com/zsh-users/zsh-autosuggestions
[3]: https://github.com/robbyrussell/oh-my-zsh/wiki/themes
[4]: https://github.com/robbyrussell/oh-my-zsh/tree/master/plugins
[5]: https://github.com/robbyrussell/oh-my-zsh
[6]: https://github.com/ggreer/the_silver_searcher
[7]: https://github.com/junegunn/fzf
[8]: http://conqueringthecommandline.com/book/ack_ag
[9]: https://github.com/junegunn/fzf/wiki/examples



