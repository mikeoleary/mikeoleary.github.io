---
layout: single
toc: true
title:  "Centos and Red Hat notes, part 3"
categories: [redhat]
tags: [redhat, centos, linux]
excerpt: "Unlike Windows Server, which I learned 'from the ground up' by reading textbooks, I've just sort of learned Linux on the job over the years. Now I'm going back to learn the basics: history, fundamentals, and things I've always glossed over." #this is a custom variable meant for a short description to be displayed on home page
---
![Red Hat logo](/assets/red-hat-logo.svg){:style="display:block; margin-left:auto; margin-right:auto"} 
<hr />
[Part 1]({% post_url 2023-12-19-rhcsa-notes-1 %})<br/>
[Part 2]({% post_url 2023-12-26-rhcsa-notes-2 %})<br/>
[Part 3]({% post_url 2024-01-10-rhcsa-notes-3 %})<br/>
<hr />
<!-- begin_excerpt -->
I've decided to take the exams for Red Hat Certified Systems Administrator (RHCSA) and Red Hat Certified Engineer (RHCE). I've been meaning to do this for 10 or 15 years! This post is just notes for myself. 

<!-- end_excerpt -->
**I usually use Ubuntu so these notes are specifically for learning CentOS.**
<br/><br/>
<hr />

### grep and find usage
#### grep command
`grep` is the main command for browsing file contents.
- `-i` is for case insensitive
- `-d` is for directories for when the input file is a directory (ie, what do you want to do with subdirectories: **read** them like a normal file (default but will throw error when it's a directory not a file), **skip** them, or **recurse** through subdirectories)
- `-r` is for recursive (equivalent to `-d recurse`)
- `-l` or `--files-with-matches` will suppress normal output; instead print the name of each input file from which output would normally have been printed.
- `-L` or `--files-wihtout-matches` Suppress normal output; instead print the name of each input file from which no  output would normally have been printed.
- `-s` or `--no-messages` Suppress error messages about nonexistent or unreadable files.
- `-q` for quiet

The first command will find all files within /etc/* that contain the text **chrony** (case insensitive)<br/>
The second will have same output, but silence the errors output by subdirectories not being skipped.
```bash
grep -li -d skip chrony /etc/*
grep -lis chrony /etc/*
```

#### find command
`find` will search for files in a directory hierarchy
- `maxdepth` means go this number of levels deep. *-maxdepth 0* means only apply the tests and actions to the command line arguments. *maxdepth 1* means in the directory you specify in command line.
- `-type` **d** is for directory, **f** is for regular file, there are others
- `-name` pattern
- `-path` pattern
- `-mtime -ctime -atime` are modified time, status last changed time, accessed time.
This command will find all regular files in `/etc` that were modified within the last 180 days and copy all of them to a directory /var/tmp/pvt
```bash
find /etc -type f -mtime +180 -maxdepth 1 -exec cp {} /var/tmp/pvt \;
```
