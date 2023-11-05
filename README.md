# TIL

This is the source code for website: https://eliasdorneles.com/til/ -- a
collection of small notable tech things [Elias](https://eliasdorneles.com) has
learned.


## Setting up locally

Install [Hugo](https://gohugo.io/installation/).

After cloning the repository, fetch the theme:

    git submodules update --init --recursive

Install the JS dependencies:

    npm install

You're ready to go!
To test locally:

    hugo serve
