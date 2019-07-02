+++
date = "2017-08-22"
title = "Redirect Test Cli"
description = "Redirect Test - A Nodejs Cli for testing redirects"
images = ["posts/shahadat-shemul-BfrQnKBulYQ-unsplash.jpg"]
tags = ["nodejs", "npm", "open source"]
categories = ["open source"]
+++

I work for an agency that runs a lot of client sites. We rarely get to build the first site to ever be served from a new domain. Since most of the work we do is taking the old outdated stuff and giving it new life, we have to make sure to bring along everyone for the new shiny ride. Of all the things that have to be done to get ready to launch, getting redirects in place and testing them is probably one of the most time-consuming. We have a great SEO expert that gives us a nice CSV of all the old and new URIs that need to be put in place. Even putting them into our Puppet system isn't too bad. The real work starts when you have upward of 200 redirects that all need to work on launch day. Now a sane person would not want to test all of them by hand, nor should they. Most of us, myself included, would test maybe 10-20% and leave it at that. I decided that wasn't good enough and since I don't have time to write these things in my day job I wrote something in my spare time.

[Redirect-test](https://github.com/cpitkin/redirect-test) is a Nodejs CLI tool that will test your redirects for you. You just give it a CSV of inputs and it will test each route and output the errors. The output format is also a CSV with the old and new URLs but also includes that status code received. If the response was a 301 but wasn't the correct URI then the CSV will have a fourth column with the response URI. The tool also allows for testing behind basic auth just in case you need to test something before it is opened to the public.

All in all, it is a simple tool but comes in really handy when there are more than a handful of redirects to test. I hope many people find it useful and can save some valuable time. Feel free to open an issue if something isn't working as expected and pull requests are always welcome.

Photo by Shahadat Shemul on Unsplash
