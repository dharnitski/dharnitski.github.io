# dharnitski.github.io

Source code of my blog https://www.contributing.md/

## Prerequisites

For MacOS:

    brew install ruby
    gem install bundler
    bundle install

## Build and run the blog locally

Script to start blog with debug info and unpublished posts:

    bundle exec jekyll serve --verbose --trace --drafts --future

Open link in Browser:

    http://127.0.0.1:4000/