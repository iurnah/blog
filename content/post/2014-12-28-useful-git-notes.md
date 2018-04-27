---
title: "Useful Git Notes"
date: 2014-12-28 
categories:
  - Git  
toc: true;
draft: false
---
1. How to solve the big headache I have: Project in my mac, project in my linux desktop, and project in the github are out of synchronous. How to check the difference and how to make them synched?

	Answer. You'll have to merge the sync your linux project files with your github project files first, then to sync your mac project file with the updated github project files.

2. How to list references in a local repository?

	```
	SilverBear:notes Rui$ git show-ref

	c6e242aa15150b4fdb620b65906ff8c8fb155e08 refs/heads/master

	c6e242aa15150b4fdb620b65906ff8c8fb155e08 refs/remotes/origin/HEAD

	c6e242aa15150b4fdb620b65906ff8c8fb155e08 refs/remotes/origin/master
	```

3. How to check the remotes?

	```
	[18][21:15:30][rui@lab]$git config --get remote.origin.url
	git@github.com:iurnah/jos-course.git
	[19][21:15:34][rui@lab]$git remote show origin #Inspecting a Remote
	 remote origin
	  Fetch URL: git@github.com:iurnah/jos-course.git
	  Push  URL: git@github.com:iurnah/jos-course.git
	  HEAD branch: master
	  Remote branch:
	    master tracked
	  Local branch configured for 'git pull':
	    master merges with remote master
	  Local ref configured for 'git push':
	    master pushes to master (up to date)
	[20][21:16:18][rui@lab]$git remote -v
	origin  git@github.com:iurnah/jos-course.git (fetch)
	origin  git@github.com:iurnah/jos-course.git (push)
	```

4. How to add multiple remotes to a single local repo? (use case: clone from mit6.828 repo, push to my github repo)

	Format: git remote add [shortname] [url]

	```
	$ git remote
	origin
	$ git remote add pb https://github.com/paulboone/ticgit
	$ git remote -v
	origin https://github.com/schacon/ticgit (fetch)
	origin https://github.com/schacon/ticgit (push)
	pb https://github.com/paulboone/ticgit (fetch)
	pb https://github.com/paulboone/ticgit (push)
	```

5. How to add a repo inside another repo? (use case: mit/lab repo clone from mit6.828, put under my github/jos-course repo, lab repo inside jos-course repo. or I clone a library in side my project repo, I need track my project as well as the library repo for library update.)

	We can use submodule of git.

	Add labs repo cloned from mit6.828 to my own repo jos-course

	```
	[rui@jos-course]$git submodule add http://pdos.csail.mit.edu/6.828/2012/jos.git labs #add directory and clone from the url
	[rui@jos-course]$git commit -m "add submodule directory"
	[rui@jos-course]$git push
	pushing submodule labs to repo jos-course.[rui@jos-course]
	pulling upstream changes from mit6.828
	checkout branchs from mit6.828
	```

	Pay attention to the following use cases:

	- clone a repo contain submodule.
	- update submodule/submodules
	- Git will by default try to update all of your submodules when you run git submodule update --remote so if you have a lot of them, you may want to pass the name of just the submodule you want to try to update.

6. How to change the name of the remote repo and update the remote url for local repo?

    see this help doc from github

7. How to transform multiple copy of documents/code to a git repo that keep all the different information. such as I have previously write app_v1.cpp. app_v2.cpp, ... to a single repo in github have only one file app.cpp that keep all the version information.