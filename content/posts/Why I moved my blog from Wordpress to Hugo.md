+++
title = 'Why I Moved My Blog From Wordpress to Hugo'
date = 2024-05-26T13:55:32-04:00
draft = false
tags = hugo,meta
+++


# Why I moved my blog from Wordpress to Hugo

Why are you making a big deal about this? You only have one article on your blog! Are you ever going to write about what you're working on, or are you just going to keep re-inventing the wheel?

Yeah... Great questions. 

I do have a counter to that last question, though. This move is exactly the kind of project I want to work on and write about. My homelab is my playground. Shaking things up is what makes this project so fun. And, as I'll explain in a bit, this move will allow me to practice some important skills when it comes to DevOps practices.

## Speed and security
I won't get into explaining what [Hugo](https://github.com/gohugoio/hugo) is in this article, but just know that it is what is called a "static site generator".  That means that I can write a blog post in markdown and run a few commands to automatically generate a website from those files. Beyond that, this site is what is called a "static site". *Static* webpages stand in contrast to standard webpages which are classified as "dynamic". 

These static pages have numerous advantages over dynamic sites. A big one for me is that they are much faster to load because there is no server-side computation going on and there's also much less work being done by the client (your web browser.) Load times are considerably faster than what I was seeing on my Wordpress site running in a Docker container. The other huge benefit of static sites is that they are less susceptible to security vulnerabilities. There is far less (if any) Javascript being executed that could be exploited by attackers. But being better than Wordpress at security is [kind of a low bar](https://en.wikipedia.org/wiki/WordPress#Vulnerabilities) anyway.

## Pipelines
Hugo can be easily integrated with continuous integration and continuous delivery (CI/CD) pipelines. This means that you can automate the process of building and deploying your website. For example, I use GitHub Actions to automatically build and deploy my site every time I push changes to my Git repository. This is a huge benefit for me, as it allows me to automate the deployment process and ensure that my website is always up to date.

## More control
With Hugo, I have more control over the look and feel of my site. I can create my own themes and layouts, and I can customize the way my content is displayed. I also have more control over the underlying code, which gives me the flexibility to add custom features and functionality. This is great for me, as I want learn more about web development anyway.

## Transparency
Both Hugo and Wordpress are open source, which is great. I have confidence that the community of passionate maintainers will keep both services free and secure.

However, Wordpress is mostly maintained by a single corporate interest, Automattic. Because of this, the future of Wordpress is slightly more questionable than Hugo.

Using Hugo also has the added benefit of making it easier to open source my site. Right now you can go to my Github page and view the entire source code for my blog. Pretty neat!

## How it Hugo fits into my PKM system
I use a PKM (personal knowledge management) system to organize my notes, articles, and other digital resources. Hugo can be integrated with my PKM system to automatically generate blog posts from my notes. This allows me to easily share my knowledge with others and to keep my blog up to date with my latest thinking. Plus, I already use Obsidian which is a natively markdown note taking application. Because of this, publishing a blog post is as easy as copying one of my notes into my Hugo repo.

## Conclusion
Overall, I'm really happy with my decision to move my blog from Wordpress to Hugo. I've noticed a significant improvement in speed and security, and I have more control over the look and feel of my site. I'm also excited about the potential for using Hugo to automate my blogging workflow and integrate it with my PKM system.

I'm also looking forward to using this move as an opportunity to learn more about DevOps practices. By using GitHub Actions to automate my deployment process, I'm getting hands-on experience with a key DevOps tool.

Of course, there are some downsides to using Hugo. It's not as easy to use as Wordpress, and there's a smaller community of users. However, I'm confident that the benefits of using Hugo outweigh the drawbacks.

If you're considering moving your blog to Hugo, I encourage you to give it a try. It's a great way to improve the speed, security, and control of your site.