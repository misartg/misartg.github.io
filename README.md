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

1. Install ruby. If you have chocolatey, `choco install ruby.install` should do. That should come with Ruby's `gem` installer as well.  

2. Install jekyll. `gem install jekyll bundler` should work for you. 

3. Clone this repo.

4. Enter this repo's root folder and run `bundle install` to install the needed dependencies.

5. Start the site with: `bundle exec jekyll serve --livereload --drafts --port 8080`. You can view the site from your browser at http://localhost:8080. You can stop the local server with `CTRL` + `C`. 

While you're developing your posts, you probably want to keep them in the `_drafts` folder so they don't get published even if pushed to your public repo. They can be navigated to and viewed in the GitHub repo itself, but they're not linked or visible from the main https://misartg.github.io page. 