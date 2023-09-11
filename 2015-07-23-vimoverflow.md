# Vimoverflow

> This is probably more no longer working, keeping it here for historical reasons.


After I read this funny article [Computer Programming To Be Officially Renamed “Googling
Stackoverflow”](http://www.theallium.com/engineering/computer-programming-to-be-officially-renamed-googling-stackoverflow/).
I get the idea to create a plugin to search stacoverflow from VIM. So I start my
journey into Vim scripting which doesn't last very long! It was hard to begin
with especially to develop a plugin that do some scarping.

<!-- more -->

## How I did it:

I used python to search using Google. And then extract the result from
Stackoverflow. With the help of 3rd party libraries it was very easy to put
things on the right track.

## Installation:

If you don't have a plugin manager I recommend
[Vundle](http://https://github.com/VundleVim/Vundle.vim).

#### Vundle 
Add the following to your `~/.vimrc` file and run `PluginInstall` in Vim.

    Plugin 'Chakerbh/vimoverflow'


## How to use it: 

All you need to do is to run:

    :VimOverflow ques

Where ques is your question

**Example:**

    :VimOverflow length of list python


    The len function can be used with a lot of types in Python - both built-in types and library types.
    >>len([1,2,3])
    3

## Fork it:

You can fork it on [Github](https://github.com/chakerbh/vimoverflow). And if you have any question feel free to ask in the comment section below.

