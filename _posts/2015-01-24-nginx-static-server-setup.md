---
layout: post
title: Setting Up A Static HTML Server with Nginx
description: "Basic Nginx setup to serve static files and rewrite routes to accomodate wishy-washy directory structures and buldpaths."
modified: 2015-1-24
tags: [nginx, server, devops, webserver]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---
Nowadays, the teams I've worked with all develop their websites using Javascript-based tools, such as Gulp and Bower. Accordingly, they deploy Node.js locally to test stuff out while developing. However, in situations where a backend isn't necessary, we prefer to deploy our websites using nginx. When push comes to shove, nginx is the current leader when it comes to performance, and since we're looking to eke every last cycle out of our production boxes, we'll go with the most performant choices in prod. That being said, however, Node projects don't always typically map over cleanly to a typical webserver config. So usually there's quite a bit of internal shuffling around and non-trivial configuration options that make even the simplest of nginx deployments like this harder than they should be. Compound this with the fact that I don't typically get the chance to deploy nginx often, and what should be an hour's work max becomes about six hours of swearing and continually reloading configuration files. 

Well, no more! This time, I'm going to document exactly what I did, and why I did it. That way, the next time I need to deploy an nginx config I'll at least have somewhere to remember how to do the basic stuff. So without further ado, let's get started!

## The Config Itself

{% highlight nginx %}
    server {
        listen 80;
        server_name example.com www.example.com xxx.xxx.xxx.xxx;
        root /home/admin/site/dist;

        rewrite ^/static/(.*)$ /$1;
        rewrite ^/sponsor.pdf$ /files/sponsor.pdf;
        rewrite ^/robots.txt$ /files/robots.txt;
        rewrite ^/favicon.ico$ /img/favicon.ico;

        location / {
            index pages/index.html;
        }

        location /register {
            try_files /pages/$uri.html /pages/$uri;
        }

        location /mentor {
            try_files /pages/$uri.html /pages/$uri;
        }
    }
{% endhighlight %}

## The Explanation
We'll go in logical sections. Breaking it down like this makes me think about it more, and helps with remembering.

### Basic Configuration

    listen 80;
    server_name example.com www.example.com xxx.xxx.xxx.xxx;
    root /home/admin/site/dist;

This is some initial global configuration. Nginx uses an inheritance model when parsing its configuration files that approximates the notion of scope in typical programming languages. Anything defined outside a set of brackets will also carry over and apply inside that set of brackets. So by that logic, anything in the root of the server block will apply to every nested block within. This makes it a good place to set up general configuration options. In this case, I've started by defining some no-nonsense global defaults:

1. listen 80; - the port to listen on.
2. server_name example.com www.example.com xxx.xxx.xxx.xxx; - This defines the hostnames that this server block will respond to. In our case, we'd like to respond to example.com, with or without the www, and our IP address. In all other cases, nginx will just silently refuse to complete the connection.
3. root /home/admin/site/dist - This defines the root path for this server block. All relative file paths when resolving requests will be calculated relative to this directory path.

### Top-level Rewrites

    rewrite ^/static/(.\*)$ /$1;
    rewrite ^/sponsor.pdf$ /files/sponsor.pdf;
    rewrite ^/robots.txt$ /files/robots.txt;
    rewrite ^/favicon.ico$ /img/favicon.ico;

Here begins the deployment-specific configuration. That first line alone took nearly an hour to figure out how to do, which is something I'm not proud of, but again, there's a reason why I'm writing this blog. All of these lines make use of the rewrite directive, which lets you tell nginx to look in different places for the result of a request, rather than naively following the relative filepath indicated by the request itself. 

I think lines 2, 3, and 4 are straightforward to understand. However, at a glance, line 1 would give me pause, so I'm going to talk about it a little bit more. In the actual HTML, all the paths to vendor JS libraries are linked under a subdirectory "/static". This is a made-up subdirectory that Node automatically resolves. On the actual filesystem, however, the directory tree looks like this:

    dist
    ├── files
    ├── img
    ├── pages
    └── vendor

Because of this, in conjunction with the root directive from before, Nginx will try to look for the directory path "/home/admin/site/dist/static", not find it, and promptly complain. So what line 1 does is make nginx disregard the /static/ part at the beginning of the relative URL, and go through the rest of the filepath looking for the asset. So, for example, "/static/vendor/bootstrap/dist/css/bootstrap.css" becomes "/vendor/bootstrap/dist/css/bootstrap.css" when nginx goes to find it in the filesystem. Piece of cake.

### Root Location, Or, How To Tell Nginx What Your Homepage Is

    location / {
        index pages/index.html;
    }

This makes use of the index directive to let Nginx know where your index page is for that route. In our case, we wish to define the index for the root, and we've told nginx to server up pages/index.html. This is an *internal redirect*, which means that the user will not see "http://example.com" turn into "http://example.com/pages/index.html" in their URL bar.

### How to Create Custom Routes Without Express.js

    location /register {
        try_files /pages/$uri.html /pages/$uri;
    }

This is the last piece of the puzzle. Using custom locations, we can where to search for files when a specific GET location is requested. Using the try_files directive, we tell nginx to search in one of various paths for the file. This directive is mainly meant for setting up more complicated resolver strategies. I've seen it used in setups where backing content would be dynamically inserted, and if not, the webmaster can easily serve a generic redirect or index page by giving it as an argument to this directive. In this case, we're using it because I wanted to experiment with the directive and because it lends a bit of variety. A small timesaving optimization is that I specified the actual path as the first argument to the directive, meaning that nginx will search (and, in this case, find) the resource immediately. "$uri" is a placeholder that turns into the actual URI requested; in this case, that would be "register". The /mentor route works in exactly the same way.

## In Conclusion
In hindsight, it's obvious that this is a rather basic nginx config. Nevertheless, it does go through a good deal of the more common directives and patterns used when building nginx configurations. Because this is a static webpage, we didn't have the opportunity to get into nginx's other area of expertise, proxying requests to other servers. Maybe I'll save that for next time. 
