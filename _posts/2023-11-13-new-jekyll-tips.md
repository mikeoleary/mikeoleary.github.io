---
layout: single
title:  "Tips I learned about Jekyll today"
categories: blogging
tags: blogging
---

This is a short post intended to take note of the changes I made to this blog today.

#### Today's learning tips

1. **Categories vs Tags**<br/>
When I started using Jekyll up until today, I didn't understand the difference. Now I do, and for the time being I think I will use both categories and tags in my posts. Maybe I will want to organize via one or the other in the future. Today I added a Categories page. [Here](https://blog.webjeda.com/jekyll-categories/) is the link I followed to teach me the basics. 

2. **Permalinks**<br/>
I knew I had to update my permalink style. I followed [this easy guide](https://ben.balter.com/jekyll-style-guide/permalinks/). Now my permalink style looks like this in my ```_config.yml```

{% highlight yaml %}
permalink: /:categories/:title/
{% endhighlight %}

**Now I know I can change this later without breaking links!** I have updated any links in between my different posts to use the ```post_url``` tag. When I read the following pro tip from [this page](https://jekyllrb.com/docs/posts/) I thought it was worth copying out again:

>Use the post_url tag to link to other posts without having to worry about the URLs breaking when the site permalink style changes.

{:start="3"}
3. **The About page**<br/>
I knew that I could create pages, I just had not got around to doing this. I now have an [About](/about) page.

#### Future plans
In future I intend to create a better home page, try out a different theme, find a way to add emojis to posts, create a few more static pages, and see if I can create some kind of word cloud similar to the right hand side of Thomas Stringer's [blog](https://trstringer.com/). But for now I'll end this post and commit my changes.
