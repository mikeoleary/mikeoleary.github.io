---
layout: post
title:  "More tips I learned about Jekyll"
categories: blogging
tags: blogging
---

### Tips I learned
For the sake of taking notes, here's what I learned about Jekyll recently

1. You should manually create the ```_drafts``` folder. I thought that would be created by default but [according to the docs](https://jekyllrb.com/docs/posts/#drafts) you create it yourself.
   >To get up and running with drafts, create a _drafts folder in your site’s root and create your first draft

2. The ```_layouts```, ```_includes``` and ```_sass``` directories are not in my site root when I set up my first site, and that's because since v 3.2 they are stored in the theme-gem by default. I am running ubuntu and when I run ```bundle info --path minima``` from the root of my site, I am pointed to a directory: ```/var/lib/gems/3.0.0/gems/minima-2.5.1```. This is where these directories are by default.

3. I think you are supposed to create (or copy) the ```_layouts``` directory into your site's root if it is not there. It says to do this [in the docs](https://jekyllrb.com/docs/step-by-step/04-layouts/#creating-a-layout).
   >Create the _layouts directory in your site’s root folder and...
   In any case, I have copied the ```_layouts``` directory from ```/var/lib/gems/3.0.0/gems/minima-2.5.1``` into my site's root.

### Still to do

1. I would still like to change my theme from minima (the default) to something else. **I want to minimize the time I spend worrying about visual formatting, HTML, CSS, anything like that**. I like the look of the theme called "Clean Blog" which, for reference, is [here](https://jekyllthemes.io/theme/startbootstrap-clean-blog-jekyll). 

2. After picking a theme I would like to customize it and then **never need to touch it again**. 

3. I would like to add tags to articles, as well as categories.

4. I would like to re-order the categories when you click on the Categories page. Currently they are ordered by age, so my oldest post's category is listed first. I think I'd like to order by number of posts within that category.

5. I would like the categories page to show a few (perhaps 3) blogs for each category, but then link to a list of *all* posts within that category.

6. I still need to learn how to use the ```{{ post.excerpt }}``` feature within Jekyll, and then go back and add these to my posts.