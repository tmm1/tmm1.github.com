---
layout: post
title: "Ruby 2.1: Method Cache"
---

For years, MRI cleared the entire VM's method cache whenever a new method was defined. In fact, method defintions were only one of [a dozen ways to clear the method cache](https://charlie.bz/blog/things-that-clear-rubys-method-cache). Earlier this year, @jamesgolick decided to improve things with a [patchset for 1.9 implementing hierarchical invalidation](http://jamesgolick.com/2013/4/14/mris-method-caches.html). @charliesome subsequently [ported and committed the patchset](http://bugs.ruby-lang.org/issues/8426) to trunk. Starting with Ruby 2.1, altering a class will only invalidate the caches for that class and its subclasses.

To provide visibility, we've exposed some basic stats about the method cache via a new `RubyVM.stat()` method. For instance, an application can measure <tt>"global invalidations per request"</tt> by comparing `RubyVM.stat(:global_method_state)` before and after every request. See [simeonwillbanks/busted](https://github.com/simeonwillbanks/busted) for some convenience methods around these new stats.

To track down where (global and non-global) invalidations are happening, Ruby 2.1 ships with a new probe: `ruby::method-cache-clear`. This can easily be used via [dtrace](https://github.com/simeonwillbanks/busted/blob/master/dtrace/probes/examples/method-cache-clear.d) or [systemtap](http://avsej.net/2012/systemtap-and-ruby-20/) to find the source of method cache invalidations in your application.

``` console
$ cat ruby_mcache.stp
probe process("/usr/bin/ruby").mark("method__cache__clear") {
    printf("%s(%d) %s %s:%d cleared `%s'\n", execname(), pid(), $$name, kernel_sring($arg2), $arg3, kernel_string($arg1))
}

$ sudo stap ruby_mcache.stp
ruby(25410) method__cache__clear lib/ruby/2.1.0/ostruct.rb:169 cleared `OpenStruct'
ruby(25410) method__cache__clear lib/ruby/2.1.0/ostruct.rb:170 cleared `OpenStruct'
```

Work on improving ruby's method cache is continuing on ruby-core. Early numbers show up to 5-10% improvements are possible with [a larger and more resilient cache](http://bugs.ruby-lang.org/issues/9262). We hope to land some improvements to ruby-trunk (for 2.2), and maybe backport them into a future 2.1 patchlevel release.
