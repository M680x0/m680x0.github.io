# Website for M68k LLVM Support

A portal to the development of M68k LLVM support, as well as other M68k releated documentations.

## Contributing to the site

### Setup

#### macOS

Using Homebrew and rbenv so we don't interfere with system Ruby. Skip or alter steps if using a different Ruby version manager.

```sh
brew install rbenv
rbenv init
```

Restart your terminal to allow `rbenv init`'s changes to take effect.

```sh
rbenv install 3.1.7
rbenv global 3.1.7
```

Install bundler and Jekyll:

```sh
gem install bundler jekyll
```

Install the gems listed in `Gemfile`:

```sh
bundle install
```

Start the Jekyll server:

```sh
bundle exec jekyll serve
```

Open the site being served:

```sh
open http://127.0.0.1:4000/
```
