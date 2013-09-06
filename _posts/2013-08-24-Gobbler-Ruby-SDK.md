---           
layout: post
title: Gobbler Ruby SDK
date: 2013-08-24 00:00:00 UTC
comments: false
categories: gobbler ruby
---

A few weeks ago I wrote and released a [Ruby
SDK](http://rubygems.org/gems/gobbler) around the publically
availavle (but not so well documented) APIs that are available at
[Gobbler](http://gobbler.com). My goals were to get more experience
with writing an SDK, as well as give people (myself included) easier
programatic access to the information that we store in Gobbler accounts.

I modeled the gem largley after Amazon's [aws-sdk](https://github.com/aws/aws-sdk-ruby)
since I often use that and like how they structured it.

The source is available at [Github](http://github.com/gobbler/gobbler)
and I will definitely look at any issues created there or pull requests
for improvements.


### Community

If you are interested in developing using the Gobbler API and/or this
SDK, please consider joining the [Gobbler Hackers](https://www.facebook.com/groups/518902654857383) Facebook group that we put together to communicate. Feel free to get in touch there or to me directly if you have any questions, issues, comments, etc..

I am a very big believer in open source software and sharing code, so
any work that goes towards the goal of an open, developer-friendly
community around Gobbler, I am all for it.


### Installation

Installation is easy, just use the standard:

{% highlight ruby %}
  gem install gobbler
{% endhighlight %}


### Usage

Here is a screencast of some basic functions of the gem:

<iframe width="640" height="480" src="//www.youtube.com/embed/ROhJqBksov0" frameborder="0" allowfullscreen="allowfullscreen">youtube video</iframe>


### Example Code

Here are the basic examples from the [README.md](https://github.com/gobbler/gobbler/blob/master/README.md)

{% highlight ruby %}
  require 'gobbler'

  ## Set up authentication and sign in
  Gobbler.config(email: "...", password: "...")

  ## Get the high-level metrics for your account
  puts Gobbler::Dashboard.list

  ## Get a list of all project names
  puts Gobbler.projects.collect(&:name)

  ## Get a list of all files in a project
  project = Gobbler.projects.first
  checkpoint = project.last_checkpoint

  checkpoint.assets.each do |asset|
    puts asset["relative_path"]
  end

  ## Get a list of all machines that you have signed into gobbler with
  puts Gobbler.machines

  ## Get a list of the cities that each of your drives was last seen in
  Gobbler.volumes.each do |volume|
    puts "#{volume.volume_name} : #{volume.city}"
  end
{% endhighlight %}
