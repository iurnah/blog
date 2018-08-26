---
title: "Setting File Permissions for Web Documents"

date: 2018-08-26
categories:
  - Web
tags:
  - Permission
  - Security 
toc: true
disable_comments: true
---
What file permissions the web documents should have in order to make development
process easier? 

The answer depends. For my use case, I want to avoid using `sudo` for copy document from the
user's home to the Apache document root directory everytime I want to update a file. 
Because the `/etc/www` directory owned by root, using sudo to copy a file into the
document root `/etc/www/html` will give the copied file "root" permission. This is both insecure and
inconvenience. What permission should I give it?

I have the following requirements at the moment:

1. Allow apache to access the documents and traverse the document root directory.
2. Do not have to use "sudo" while working on the website documents.
3. CD scripts can work properly without using setuid.

For my blog, it is in `/var/www/html/blog`. The permissions look like this
before getting changed:

```
drwxr-xr-x  7 rui  rui  4096 Apr 23 05:48 .
drwxr-xr-x  3 root root 4096 Apr 20 04:58 ..
-rw-r--r--  1 root root 1859 Apr 23 06:13 404.html
drwxr-xr-x  2 root root 4096 Apr 23 05:28 about
drwxr-xr-x  9 root root 4096 Apr 23 06:13 categories
drwxr-xr-x  2 root root 4096 Apr 23 05:28 css
-rw-r--r--  1 root root    0 Apr 23 05:23 .gitkeep
-rw-r--r--  1 root root   87 Apr 23 05:23 .htaccess
-rw-r--r--  1 root root 3510 Apr 23 06:13 index.html
-rw-r--r--  1 root root 7960 Apr 23 06:13 index.xml
drwxr-xr-x  7 root root 4096 Apr 23 05:28 post
-rw-r--r--  1 root root 4774 Apr 23 06:13 sitemap.xml
drwxr-xr-x 12 root root 4096 Apr 23 06:13 tags
```

As you can see that the `blog` directory is owned by the user "rui", but the files inside the
directory is owned by "root". After reading the answers in [serverfault][1], I would like to 
change the file permissions in this directory and all the subdirectories to the following:

```
drwxr-xr-x  7 rui  rui  4096 Apr 23 05:48 .
drwxr-xr-x  3 root root 4096 Apr 20 04:58 ..
-rw-r--r--  1 rui www-data 1859 Apr 23 06:13 404.html
drwxr-xr-x  2 rui www-data 4096 Apr 23 05:28 about
drwxr-xr-x  9 rui www-data 4096 Apr 23 06:13 categories
drwxr-xr-x  2 rui www-data 4096 Apr 23 05:28 css
-rw-r--r--  1 rui www-data    0 Apr 23 05:23 .gitkeep
-rw-r--r--  1 rui www-data   87 Apr 23 05:23 .htaccess
-rw-r--r--  1 rui www-data 3510 Apr 23 06:13 index.html
-rw-r--r--  1 rui www-data 7960 Apr 23 06:13 index.xml
drwxr-xr-x  7 rui www-data 4096 Apr 23 05:28 post
-rw-r--r--  1 rui www-data 4774 Apr 23 06:13 sitemap.xml
drwxr-xr-x 12 rui www-data 4096 Apr 23 06:13 tags
```

I have done the following:

```
chown -R bob blog/
chgrp -R www-data blog/
chmod g+s blog/

find /var/www/html -type d -exec chmod 750 {} +
find /var/www/html -type f -exec chmod 640 {} +
```

Now the permissions are looks like:

```
drwxr-s---  7 rui www-data  4096 Aug 26 20:02 .
drwxr-s---  3 rui www-data  4096 Apr 20 04:58 ..
-rw-r-----  1 rui www-data  1941 Aug 26 20:04 404.html
drwxr-x---  2 rui www-data  4096 Apr 23 05:28 about
drwxr-x--- 14 rui www-data  4096 Aug 26 19:29 categories
drwxr-x---  2 rui www-data  4096 Apr 23 05:28 css
-rw-r-----  1 rui www-data     0 Apr 23 05:23 .gitkeep
-rw-rw-r--  1 rui www-data    87 Apr 23 04:19 .htaccess
-rw-r-----  1 rui www-data  4649 Aug 26 20:04 index.html
-rw-r-----  1 rui www-data 13720 Aug 26 20:04 index.xml
drwxr-x---  8 rui www-data  4096 Aug 26 19:29 post
-rw-r-----  1 rui www-data  7051 Aug 26 20:04 sitemap.xml
drwxr-x--- 16 rui www-data  4096 Aug 26 19:29 tags
```

The above settings look good enough for me and the site is working properly. In the future, I have
to get into web security model and improve upon this simple file permission configuration with
strict security consideration.

[1]: https://serverfault.com/questions/357108/what-permissions-should-my-website-files-folders-have-on-a-linux-webserver
