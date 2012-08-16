---
layout: post
title: "Blogging with Jekyll & Github Pages"
description: ""
category: 
tags: []
---
{% include JB/setup %}
#####The Basics
**Jekyll**

From their README on Github,
> Jekyll is a simple, blog aware, static site generator
Essentially, you define a few styles and your blog posts. Then you run Jekyll on those files and it outputs static html to a `_sites` folder. You can then use any webserver to host those files

**Github Pages**

Github offers free hosting for any static content you create. To sign up for one, log in to Github and create a new repository called *yourusername*.github.com. Then push your content to the new repository and it will be available at [http://*yourname*.github.com](http://yourname.github.com).

#####Why Jekyll & Github pages?

When I went looking for a blogging platform I settled on Jekyll and Github Pages for a few reasons.
+ It's just text. This means it's super fast because the server only has to serve static HTML.
+ Another excuse to use git and Github. I've been using more and more git lately and this was another way to hone my skills.
+ It's free! Github hosts Github Pages for free. Anybody can throw content up there or start blogging.
+ It's portable. If I need/want to move the content somewhere else, it's just a bunch of markdown text that I can move to another blogging platform. You can even host the content on your own server if you wish.

#####Getting started
The quickest way I found to get up and running was to use jekyll-boostrap.

1. Follow [the steps outlined here](http://jekyllbootstrap.com/usage/jekyll-quick-start.html) to get up and running with jekyll. This creates a basic blog with a few dummy posts using the default twitter bootstrap theme.
1. Update `_config.yml` with your information.
1. Push the code back to github.
>`git push origin master`

#####Customizing the theme
If you're like me, you find the default theme a little boring. To update the theme edit the following files
+ `_includes/themes/twitter/default.html` - This is the master theme file. Most of the markup and styling will go here, including the header and footer.
+ `assets/themes/twitter/css/style.css` - This is where all of your custom styles should go.
+ `index.html` - This is where your homepage markup is located. I customized this page to show the first 5 blog posts instead of a list of all my posts using the code below:
{% for post in paginator.posts %}
  <h2 class="title"><a href="{{ post.url }}">{{ post.title }}</a></h2>
  <p class="author">
    <span class="date">{{ post.date | date: "%d %B %Y" }}</span>
  </p>
  <div class="content">
    {{ post.content }}
  </div>
{% endfor %}
