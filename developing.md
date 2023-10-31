1. I use [rbenv][rbenv] to manage Ruby versions.
2. `rbenv global 3.1.3 && rbenv init` (or whatever the latest Ruby version is).
3. `bundle install`.
4. `bundle exec jekyll serve`.
5. `gem uninstall bundler && gem install bundler` if 3. is causing issues.

[rbenv]: https://github.com/rbenv/rbenv
