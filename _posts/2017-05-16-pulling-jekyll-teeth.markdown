---
layout: post
title: "Pulling Jekyll teeth on the Windows Subsystem for Linux"
date: 2017-05-16 16:00:00 +0100
categories: jekyll hyde ubuntu
---
As I use more open source frameworks and tools, I've found that I've been writing down documentation that isn't just useful to business. So I decided to start a new dev blog. I checked out Wordpress and I've used Blogger before but I decided that static pages were the way to go, using the excellent [GitHub pages][github-pages]. 

GitHub will host your pages for you, all you have to do is go through the steps on the [GitHub pages][github-pages] site, clone the repo locally, add your web pages, push them back up and off you go. In the simplest form, that's very neat but it suffers from the problem that the blog posts are all mixed up with the markup. If you want to change the look and feel of your site, you're stuck.

Enter Jekyll
-------------

Jekyll is a Ruby that transforms your markdown blog text into lovely static HTML pages. You create your blog post, run it through Jekyll, check it looks OK and then push the result to GitHub. It's a great idea, I wanted to use it.

I'm new to Linux so I thought I'd use this as a way of learning more about the 
[Windows Subsystem for Linux][microsoft-wsl] (WSL). Jekyll isn't officially supported on Windows, so I thought it was a good omen to fire up the WSL.

The Jekyll website was so sure of itself:

> Get up and running in _seconds_.

Nonsense. Getting Jekyll to work was a hateful experience. Before you jump on your hate keyboard to point out that a lot of the problem I had were Ubuntu based, I didn't know that when I started and Jekyll failed to tell me.

Pre-requisites
---------------

- I already have WSL installed on my machine [instructions][windows-wsl]. It uses Ubuntu 14.04.4.
- I have already gone through the [Github Pages][github-pages] and cloned locally.

Installing Jekyll
------------------
Naively, I put in the first command: 

{% highlight bash %}
$ gem install jekyll bundler
ERROR:  While executing gem ... (Gem::FilePermissionError)
    You don't have write permissions into the /var/lib/gems/1.9.1 directory.
{% endhighlight %}

Firstly, I notice that it's trying to install to the gems directory of 1.9.1. Is that the Ruby version or the gems version?I went hunting and found that I didn't have the [requirements](https://jekyllrb.com/docs/installation/#requirements):

- Ruby 2.0+
- RubyGems
- Make
- GCC

Ah, so my Linux distro isn't complete enough. I start off by installing [GCC](https://gcc.gnu.org/) and Make.

{% highlight bash %}
sudo apt-get install gcc
sudo apt-get install make
{% endhighlight %}

It seems odd that Jekyll needs a compiler and make tool to work. Jekyll assured me that this is easy, so I kept going. So, I noticed that I needed Ruby 2.0, so what version do I have?

{% highlight bash %}
ruby -v
{% endhighlight %}

Which gives:

{% highlight bash %}
ruby 1.9.3p484 (2013-11-22 revision 43786) [x86_64-linux]
{% endhighlight %}

OK, so that's out of date. I'll update:

{% highlight bash %}
$ sudo apt-get install ruby
Reading package lists... Done
Building dependency tree
Reading state information... Done
ruby is already the newest version.
The following packages were automatically installed and are no longer required:
  fgetty os-prober
Use 'apt-get autoremove' to remove them.
0 to upgrade, 0 to newly install, 0 to remove and 144 not to upgrade.
{% endhighlight %}

Already the newest version, eh? I think not! It's over to the [Ruby Install][ruby-install] page. However Ubuntu's `apt-get` will only get you as far as version 1.9.3. So how do you get Ruby 2 on Ubuntu 14.04. That sounds like a Stack Overflow question, [which it is](http://stackoverflow.com/q/26595620/328730).

Enter rbenv
----------------------------

Silly me for thinking that `apt-get` would install Ruby 2. No, I have to use [rbenv][ruby-rbenv]. You have to install a load of pre-requisites (some I already had) and then `git clone` the `rbenv` repository into a folder, add PATH variables and so on.

[From Stack Overflow answer:](http://stackoverflow.com/a/26595869/328730)
{% highlight bash %}
sudo apt-get update
sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev

git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL

git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
exec $SHELL

rbenv install --verbose 2.4.1
rbenv global 2.4.1
ruby -v
{% endhighlight %}

`rbenv install` took several minutes. I added the `--verbose` switch so that I could see it going (others [had thought it had hung](http://stackoverflow.com/q/23944406/328730)). The last command `ruby -v` gives me `ruby 2.4.1p111 (2017-03-22 revision 58053) [x86_64-linux]`, which is a relief!

It's important to remember that I'm just trying to get Jekyll to run at this point. This is a pre-requisite to a pre-requisite. I was tempted to go back to writing HTML at this point. I'm pretty productive in that, I wrote my first HTML page at the end of 1995. I'm not quite spitting teeth at this point but I'd decided that I was going to blog all this nonsense. Not the first post I wanted to write but that's blogging for you.

Let's try Jekyll again
-------------------

We have the pre-requisites installed, now to try installing Jekyll and [bundler][ruby-bundler]. Bundler is a way to manage your Ruby gems, a Ruby gem being a library. It works off a Gemfile that greatly simplifies the inclusion and management of Ruby gems.

{% highlight bash %}
$ gem install jekyll bundler
$ jekyll new my-awesome-site
{% endhighlight %}

This worked! I now had Jekyll ready to go. To build and run the site, go to the directory where your Jekyll site is and type:

{% highlight bash %}
$ jekyll serve
{% endhighlight %}

Theming
--------

Jekyll doesn't theme like other platforms. Rather than download a theme into a theme folder, you replace most of the files with theme files. I grabbed the popular [Hyde](https://github.com/poole/hyde) and replaced many of the files. You have to be careful with the `_config.yml` and `Gemfile` as these will have settings specific to the theme. I ended up with mash of Gemfile from Hyde and _config from the original new Jekyll site. It was a right mess. Rather than doing Jekyll new, I recommend cloning an existing theme into your GitHub pages folder.

Once you have the theme down, make sure you use bundler to get the gems that the theme needs.

{% highlight bash %}
$ bundler
{% endhighlight %}

When that completes, run:

{% highlight bash %}
$ jekyll serve
{% endhighlight %}

It will tell you the port it's chosen. On WSL, you need to remember that it shares ports with Windows, so you might get a port conflict if you have other sites running elsewhere.

[github-pages]: https://pages.github.com/
[microsoft-wsl]: https://msdn.microsoft.com/en-us/commandline/wsl/install_guide
[ruby-install]: https://www.ruby-lang.org/en/documentation/installation/#apt
[ruby-rbenv]: https://github.com/rbenv/rbenv
[ruby-bundler]: http://bundler.io/