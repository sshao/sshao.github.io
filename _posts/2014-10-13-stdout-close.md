---
layout: post
title: STDOUT.close
---

Closing `STDOUT` and then creating another ruby process has some 
interesting behaviors across different versions and 
implementations of ruby. Try the following on MRI 1.8, 
MRI 2.x, and Rubinius 2.x:

**`close_and_redirect.rb`**
{% highlight ruby %}
STDERR.puts "STDOUT closed?: #{STDOUT.closed?}"
STDERR.puts "sleeping... try lsof -p #{$$}\n"
sleep 10

STDOUT.close
STDERR.puts "STDOUT closed?: #{STDOUT.closed?}"
STDERR.puts "sleeping... try lsof -p #{$$}\n"
sleep 10

exec "ruby", "test_stdout.rb"
{% endhighlight %}

**`test_stdout.rb`**
{% highlight ruby %}
STDERR.puts "STDOUT closed?: #{STDOUT.closed?}"
STDERR.puts "sleeping... try lsof -p #{$$}"
sleep 10

puts "STDOUT says hello"
{% endhighlight %}

To be expanded later....
