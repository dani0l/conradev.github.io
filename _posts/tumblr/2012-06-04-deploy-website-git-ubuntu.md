---
layout: post
title: Deploy a website using git in Ubuntu
tags: 
---

This is a follow up to my [previous post](http://kramerapps.com/blog/post/22551999777/flask-uwsgi-nginx-ubuntu) on how to set up a Flask web app with nginx and uWSGI. This post will run down how to set up a git-based deployment system for that web app. Git is pretty popular for deployment, and is used in services like [Heroku](https://devcenter.heroku.com/articles/git). Heroku is great, but I prefer to have full control over my server, and roll my own stuff, all while maintaining the same ease of use.

## Repository setup

To continue from my previous example, the website is deployed in `/srv/www/helloworld`. This tutorial should work with pretty much any website, but it will still focus on Flask based ones.

First, if you have not already done so, install git.

``` bash
sudo apt-get install git
```

Next, put your website under version control with git

``` bash
cd /srv/www/helloworld
git init
git add application.py
git commit -m 'First commit'
```

A gitignore file is also a good idea. For a Flask app, put the following in `.gitignore`

```
env
*.pyc
```

and to add it to the repository

``` bash
git add .gitignore
git commit -m 'Added .gitignore file'
```

Now we are going to create a bare 'hub' repository that will act as an intermediary between the active deployment folder and the rest of the world.

``` bash
sudo mkdir -p /srv/git
sudo chown -R $USER:$GROUP /srv/git
cd /srv/git
git clone --bare /srv/www/helloworld
cd helloworld.git
git remote rm origin
```

Next, add the hub as a remote for the deployment repository

``` bash
cd /srv/www/helloworld
git remote add hub /srv/git/helloworld.git
```

## Add the magic

To automatically update the deployment repository when changes are pushed to the hub, we are going to use git hooks. Git hooks are scripts that are executed after certain changes occur within the repository.

To begin, add the following to `/srv/git/helloworld.git/hooks/post-update`

``` bash
#!/bin/sh
echo
echo "Pulling changes into deployment repository"
echo
git --git-dir /srv/www/helloworld/.git --work-tree /srv/www/helloworld pull hub master
```

and make sure that it is executable

``` bash
chmod 755 /srv/git/helloworld.git/hooks/post-update
```

This `post-update` script updates the deployment repository when the hub is updated. Next, we are going to setup the reverse, a hook to update the hub when changes are committed in the deployment directory (this should be rare, but just in case). Add the following to `/srv/www/helloworld/.git/hooks/post-commit`

``` bash
#!/bin/sh
echo
echo "Pushing changes to hub"
echo
git push hub
```

and add make it executable

``` bash
chmod 755 /srv/www/helloworld/.git/hooks/post-commit
```

## Further modification

If you went ahead and tested the deployment system, you'd find that the changes aren't reflected in your browser yet. uWSGI is still using the old compiled python files, and did not notice the changes to the directory. To get uWSGI to notice the changes, you have to reload it every time you push changes. A reload command can be added to the git hook, but it isn't as easy as it sounds. You need root privileges to reload uWSGI, and when you push changes to the server, you are most likely logging in as your own personal user. The solution to this problem is to create a setuid binary that reloads uWSGI.

First, install the Ubuntu `build-essential` package

``` bash
sudo apt-get install build-essential
```

Next, put the following in `~/reload_uwsgi.c`

``` c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    setuid(0);
    system("/sbin/initctl reload uwsgi");
    return 0;
}
```

Now to compile, install and configure the binary

``` bash
cd ~
sudo gcc reload_uwsgi.c -o /usr/local/bin/reload_uwsgi
rm reload_uwsgi.c
sudo chown root:root /usr/local/bin/reload_uwsgi
sudo chmod 4755 /usr/local/bin/reload_uwsgi
```

And lastly add this executable to our git hooks, which should look like the following

`/srv/git/helloworld.git/hooks/post-update:`

``` bash
#!/bin/sh
echo
echo "Pulling changes into deployment repository"
echo
git --git-dir /srv/www/helloworld/.git --work-tree /srv/www/helloworld pull hub master
/usr/local/bin/reload_uwsgi
```

`/srv/www/helloworld/.git/hooks/post-commit:`

``` bash
#!/bin/sh
echo
echo "Pushing changes to hub"
echo
git push hub
/usr/local/bin/reload_uwsgi
```

## Testing it out

Clone the repository on your local machine

``` bash
git clone user@server:/srv/git/helloworld.git
```

Make some changes, commit them to master, and push them to the server

``` bash
vim application.py
git commit -am 'Made some changes!'
git push origin master
```

and voila! The change should be reflected on the website.

Discuss on Hacker News [here](http://news.ycombinator.com/item?id=4067056).