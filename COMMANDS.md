# Commands

Here are some useful commands for working with Jekyll.

## jekyll-compose cheatsheet

Repo: <https://github.com/jekyll/jekyll-compose>

New page: `bundle exec jekyll page "My New Page"`

New post: `bundle exec jekyll post "My New Post"`

Rename post: `bundle exec jekyll rename _posts/2014-01-24-my-new-draft.md "My New Post"`

New draft: `bundle exec jekyll draft "My new draft"`

Rename draft: `bundle exec jekyll rename _drafts/my-new-draft.md "My Renamed Draft"`

Publish draft: `bundle exec jekyll publish _drafts/my-new-draft.md`

Unpublish post: `bundle exec jekyll unpublish _posts/2014-01-24-my-new-draft.md`

## Jekyll Cheatsheet

Install Jekyll: `gem install jekyll`

Install gems:  `bundle install`

Testing locally: `bundle exec jekyll serve`

Run on different port: `bundle exec jekyll serve --port 4001`