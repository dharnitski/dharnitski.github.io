# dharnitski.github.io

Source code of my blog <https://blog.dharnitski.com/>

## Prerequisites

For MacOS:

    brew install ruby
    brew install rbenv
    rbenv init
    sudo gem install bundler
    bundler install

## Build and run the blog locally

Script to start blog with debug info and unpublished posts:

    bundle exec jekyll serve --verbose --trace --drafts --future

Open link in Browser:

    http://127.0.0.1:4000/

## Start with docker

    docker run -p 4000:4000 -v $(pwd):/site bretfisher/jekyll-serve