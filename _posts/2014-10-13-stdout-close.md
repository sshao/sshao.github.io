---
layout: post
title: STDOUT.close
---

Closing `STDOUT` and then creating another ruby process has some
interesting behaviors across different versions and
implementations of ruby. Consider the following two files:

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

## MRI 2.0
{% highlight console %}
$ ruby -v close_and_redirect.rb
ruby 2.0.0p481 (2014-05-08 revision 45883) [universal.x86_64-darwin13]
STDOUT closed?: false
sleeping... try lsof -p 59100
{% endhighlight %}
{% highlight console %}
$ lsof -p 59100 | grep 1u
ruby  59100 sshao  1u  CHR 16,2  0t866 773 /dev/ttys002
{% endhighlight %}
{% highlight console %}
$ (continued) ruby -v close_and_redirect.rb
STDOUT closed?: true
sleeping... try lsof -p 59100
{% endhighlight %}
{% highlight console %}
$ lsof -p 59100 | grep 1u
ruby  59100 sshao  1u  CHR 16,2  0t917 773 /dev/ttys002
# stdout is still open!
{% endhighlight %}
{% highlight console %}
$ (continued) ruby -v close_and_redirect.rb
STDOUT closed?: false
sleeping... try lsof -p 59100
STDOUT says hello
{% endhighlight %}
{% highlight console %}
$ lsof -p 59100 | grep 1u
ruby  59100 sshao  1u  CHR 16,2  0t969 773 /dev/ttys002
{% endhighlight %}

stdout (fd 1) is never actually closed -- even when ruby says it is.

## RBX 2.x
{% highlight console %}
$ ruby -v close_and_redirect.rb
rubinius 2.2.10 (2.1.0 bf61ae2e 2014-06-27 JI) [x86_64-darwin13.4.0]
STDOUT closed?: false
sleeping... try lsof -p 30091
{% endhighlight %}
{% highlight console %}
$ lsof -p 30091 | grep 1u
rbx 30091 sshao  1u  CHR 16,2  0t5050  773 /dev/ttys002
{% endhighlight %}
{% highlight console %}
$ (continued) ruby -v close_and_redirect.rb
STDOUT closed?: true
sleeping... try lsof -p 30091
{% endhighlight %}
{% highlight console %}
$ lsof -p 30091 | grep 1u
# stdout is closed
{% endhighlight %}
{% highlight console %}
$ (continued) ruby -v close_and_redirect.rb
ERROR: the VM is exiting improperly running test_stdout.rb
intended operation: :exception
associated value: nil
destination scope: unknown
An exception occurred

    closed stream (IOError)

Backtrace:
  IO#fileno at kernel/common/io.rb:1560
  IO.setup at kernel/common/io.rb:977
  IO(File)#initialize at kernel/common/io.rb:995
  File#initialize at kernel/common/file.rb:1142
  Class#new at kernel/alpha.rb:94
  IO.open at kernel/common/io.rb:624
  Rubinius::Loader#write_last_error at kernel/loader.rb:794
  Rubinius::Loader#main at kernel/loader.rb:854
{% endhighlight %}

Unlike MRI 2.0, Rubinius *actually* closes stdout -- and propagates its
closed status to the exec'd process, which fails in `Rubinius::Loader` due
to the closed stream.

## MRI 1.8
{% highlight console %}
$ ruby -v close_and_redirect.rb
ruby 1.8.7 (2012-10-12 patchlevel 371) [i686-darwin13.4.0]
STDOUT closed?: false
sleeping... try lsof -p 59166
{% endhighlight %}
{% highlight console %}
$ lsof -p 59166 | grep 1u
ruby  59166 sshao  1u  CHR 16,1 0t3231295  771 /dev/ttys001
{% endhighlight %}
{% highlight console %}
STDOUT closed?: true
sleeping... try lsof -p 59166
{% endhighlight %}
{% highlight console %}
$ lsof -p 59166 | grep 1u
# stdout is closed!
{% endhighlight %}
{% highlight console %}
STDOUT closed?: false
sleeping... try lsof -p 59166
{% endhighlight %}
{% highlight console %}
$ lsof -p 59166 | grep 1u
# stdout is still closed!
{% endhighlight %}

Although `lsof` shows us that stdout is closed in the last process
(after the exec), ruby tells us that `STDOUT.closed?` is false! Furthermore,
the `puts "STDOUT says hello"` is never output.
