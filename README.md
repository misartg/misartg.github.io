# misartg.github.io
For our University of Arizona's MIS Academic and Research Technologies Group GitHub Pages site.
Our site is available at https://misartg.github.io/

### About ###

See more about about us at https://misartg.github.io/about/

### Site setup ###

Our GitHub Pages site was setup with [Jeykll](https://jekyllrb.com/), using the default minima theme. 

We also use the [jekyll-gist](https://github.com/jekyll/jekyll-gist) plugin.

We added the callouts based on Cody Clark's [callout example page](https://cody-clark.github.io/2017/07/25/my-example-post.html) and [his github repo](https://github.com/cody-clark/cody-clark.github.io/blob/master/_posts/2017-07-25-my-example-post.md). It made it much simpler and made me more confident in modifying for our use. Thanks Cody!

I think Cody's work was based on work by [Tom Johnson](https://idratherbewriting.com/aboutme/) in his [Jekyll Documentation Theme](https://github.com/tomjoht/documentation-theme-jekyll), which appears interesting and excellent. 

#### Local development environment setup ####

You can use [Jeykll](https://jekyllrb.com/) to run this site locally, allowing you to develop easily. 

1. Install ruby. ~~If you have chocolatey, `choco install ruby.install` should do. That should come with Ruby's `gem` installer as well.~~ The [choco installer doesn't have everything you'll need](https://github.com/tmm1/http_parser.rb/issues/55) to do this. Instead, [get the 64-bit Ruby+devkit installer and install it manually](https://rubyinstaller.org/downloads/). 

2. Install jekyll and bundler. `gem install jekyll bundler` should work. Wait one forever. 

3. Clone this repo.

4. Enter this repo's root folder and run `bundle install` to install the needed dependencies.

5. Start the site in auto-refresh mode and "what-if" publishing of draft posts with: `bundle exec jekyll serve --livereload --drafts --port 8080`. You can view the site from your browser at http://localhost:8080. You can stop the local server with `CTRL` + `C`. 

While you're developing your posts, you probably want to keep them in the `_drafts` folder so they don't get published even if pushed to your public repo. The draft posts can be navigated to and viewed in the GitHub repo itself, but they're not linked or visible from the https://misartg.github.io GitHub page. 
