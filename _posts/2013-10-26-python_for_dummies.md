---
layout: post
title: Python for dummies
date: 2013-10-26 14:39
categories: jekyll update
---

Virtualenv creates a completely separated sandbox for python and it's tools.

### Create a virtualenv

		python virtualenv.py /path/to/work/directory/.env

### Activation

		source /path/to/work/directory/.env/bin/activate

### Deactivation

		source /path/to/work/directory/.env/bin/deactivate
		

### Quickly create simple HTTP Server

Navigate to directory you want to serve and run the following inside, if you don't put port number, 80 will be default as usual.
 
	python -m SimpleHTTPServer 8000 