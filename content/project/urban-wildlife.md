+++
tags = []
title = "Urban Wildlife Alliance"
summary = "A webserver and website building project involving a static site generator and scripting"
+++

This project involved configuring a webserver and a publishing platform to be used for a graduate school project.

Technologies used:

* Debian
* Google Firebase
* Google Domains
* Hugo
* Bash scripting
* Let's Encrypt
* Nginx
* Rsync

[Urban Wildlife Alliance website](https://urbanwildlifealliance.org)

I was requested to help assemble a website that could be used to publish posts about urban wildlife and our coexistence with it. It needed to be easily securable and also provide an easy method for publishing. I selected Hugo as the platform for publishing as securing a static website is much easier than securing a dynamic website. I also chose Nginx for it's out of the box speed and low resource usage which is helpful for the small webserver it sits on. You never know if a post might go viral!

I also needed to make sure it was going to be easy to publish. The structure of Hugo is quick to learn but there is a learning curve with Markdown. I automated website updating by writing a Bash script that runs as a cron task to check for the presence of certain files. These files are used to issue a command to the script to perform a task such as update the website or generate a test copy of the website. This made it easy for the website owner to update the website as they only had to create an empty file in their SFTP client instead of learning how to publish the website themselves.

Just because the website was low key didn't mean that I took a low key approach. I wanted something that fit what the customer wanted as well as something I could learn from. I decided to leverage Google's Firebase and domain services, and in general staying entirely within the Google ecosystem for anything needed. The working copy of the site is stored on a Debian server and then either uploaded to Firebase with Google's CLI tool or Rsync'd to a web server for testing.
