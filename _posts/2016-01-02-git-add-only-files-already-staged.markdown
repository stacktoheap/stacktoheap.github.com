---
layout: post
title: "How to git add only files that have already been staged"
excerpt: "Adding only those files already staged, back to the index with new changes"
reading_time: "3 mins"
date: 2016-01-02 21:42
comments: true
categories: [git]
tags: [restage, git-index]
---

Git has many shortcuts that you can use to make yourself more productive. One of the little known (and simple!) features is regarding restaging files that have already been staged.

Let's create a simple repository with three files to setup an example:

{% highlight bash %}
git init .
touch file1 file2 file3
git add .
git commit -am "Add files"
{% endhighlight %}

All right. With the initial commit done, let's make changes to these files, and add the changes to the index:

{% highlight bash %}
echo "file1" > file1
echo "file2" > file2
git add file1 file2
git status
#On branch master
#Changes to be committed:
#  (use "git reset HEAD <file>..." to unstage)
#
# modified:   file1
# modified:   file2
{% endhighlight %}

So, `file1` and `file2` have been changed, and added to the index. Let's say we now make changes to `file1` and `file3`:

{% highlight bash %}
echo "file1" >> file1
echo "file3" > file3
git status
#On branch master
#Changes to be committed:
#  (use "git reset HEAD <file>..." to unstage)
#
# modified:   file1
# modified:   file2
#
#Changes not staged for commit:
#  (use "git add <file>..." to update what will be committed)
#  (use "git checkout -- <file>..." to discard changes in working directory)
#
# modified:   file1
# modified:   file3
{% endhighlight %}

`file1` and `file2`, with their initials changes, are in the index. New changes to `file1` are to be added, along with changes to `file3`.

So with our contrived example all setup, we can actually come to the main material of the post. Suppose you want to add only files that have already been staged, but have new changes in the working directory now. Files like `file1`. Git has an handy shortcut for this, but you have to look beyond `git add`. All that is needed to "restage" files in git is the following command:

{% highlight bash %}
git update-index --again
git status
#On branch master
#Changes to be committed:
#  (use "git reset HEAD <file>..." to unstage)
#
# modified:   file1
# modified:   file2
#
#Changes not staged for commit:
#  (use "git add <file>..." to update what will be committed)
#  (use "git checkout -- <file>..." to discard changes in working directory)
#
# modified:   file3
{% endhighlight %}

From the docs, this

{% blockquote %}
Runs git update-index itself on the paths whose index entries are different from those from the HEAD commit.
{% endblockquote %}

Which is exactly what we want in this scenario. Hope this nice little command makes you more productive. Git has lots of these unused little gems. Git is a complex beast, yes, but some of these need to be discovered and used.

For those who add aliases for these things, here you go:

{% highlight bash %}
git config --global alias.restage "update-index --again"
{% endhighlight %}
