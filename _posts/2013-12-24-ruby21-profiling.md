---
layout: post
title: "Ruby 2.1: Profiling Ruby"
---

Ruby 2.1 is shipping with `rb_profile_frames()`, a new C-api for fetching ruby backtraces. The api performs no allocations and adds minimal cpu overhead making it ideal for profiling, even in production environments.

I've implemented a [sampling callstack profiler for 2.1][1] called stackprof with this API, using techniques popularized by [gperftools][2] and [@brendangregg][3]. It works remarkably well, and provides incredible insight into the execution of your code.

For example, I recently used `StackProf::Middlware` on one of our production github.com unicorn workers. The resulting profile is analyzed using `bin/stackprof`:

``` console
$ stackprof data/stackprof-cpu-4120-1384979644.dump --text --limit 4
==================================
  Mode: cpu(1000)
  Samples: 9145 (1.25% miss rate)
  GC: 448 (4.90%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
       236   (2.6%)         231   (2.5%)     String#blank?
       546   (6.0%)         216   (2.4%)     ActiveRecord::ConnectionAdapters::Mysql2Adapter#select
       212   (2.3%)         199   (2.2%)     Mysql2::Client#query_with_timing
       190   (2.1%)         155   (1.7%)     ERB::Util#html_escape
```

Right away, we see that 2.6% of cpu time in the app is spent in `String#blank?`.

Let's zoom in for a closer look:

``` console
$ stackprof data/stackprof-cpu-4120-1384979644.dump --method 'String#blank?'
String#blank? (lib/active_support/core_ext/object/blank.rb:80)
  samples:   231 self (2.5%)  /    236 total (2.6%)
  callers:
     112  (   47.5%)  Object#present?
  code:
                                  |    80  |   def blank?
  187    (2.0%) /   187   (2.0%)  |    81  |     self !~ /[^[:space:]]/
                                  |    82  |   end
```

We see that half the calls into `blank?` are coming from `Object#present?`.

As expected, most of the time in the method is spent in the regex on line 81. I noticed the line could be optimized slightly to [remove the double negative][5]:

``` diff
   def blank?
-    self !~ /[^[:space:]]/
+    self =~ /\A[[:space:]]*\z/
   end
```

That helped, but I was curious where these calls were coming from. Let's follow the stack up and look at the `Object#present?` callsite:

``` console
$ stackprof data/stackprof-cpu-4120-1384979644.dump --method 'Object#present?'
Object#present? (lib/active_support/core_ext/object/blank.rb:20)
  samples:    12 self (0.1%)  /    133 total (1.5%)
  callers:
      55  (   41.4%)  RepositoryControllerMethods#owner
      31  (   23.3%)  RepositoryControllerMethods#current_repository
  callees (121 total):
     112  (   92.6%)  String#blank?
       6  (    5.0%)  Object#blank?
       3  (    2.5%)  NilClass#blank?
  code:
                                  |    20  |   def present?
  133    (1.5%) /    12   (0.1%)  |    21  |     !blank?
                                  |    22  |   end
```

So `Object#present?` is almost always called on String objects (92.6%), with the majority of these calls coming from two helper methods in `RepositoryControllerMethods`.

The callers in `RepositoryControllerMethods` appeared quite simple, but after a few minutes of staring I discovered the fatal mistake causing repeated calls to `present?`:

``` diff
    def owner
-    @owner ||= User.find_by_login(params[:user_id]) if params[:user_id].present?
+    @owner ||= (User.find_by_login(params[:user_id]) if params[:user_id].present?)
    end
```

This simple 2 byte change removed 40% of calls to `String#blank?` in our app. To further minimize the cost of `String#blank?` in our app, we switched to @samsaffron's [pure C implementation in the fast_blank gem][6].

The end result of all these optimizations was a dramatic reduction in cpu time (i.e. [without idle time][4]) spent processing requests:

<center>
![](https://f.cloud.github.com/assets/2567/1807546/e5990042-6cf6-11e3-9599-58f1e662cb0f.png)
</center>

With 2.1 and [stackprof][1], it's easier than ever to make your ruby code fast. Try it today!

[1]: https://github.com/tmm1/stackprof
[2]: https://code.google.com/p/gperftools/
[3]: http://dtrace.org/blogs/brendan/2011/12/16/flame-graphs/
[4]: http://tmm1.net/ruby21-process-clock_gettime/
[5]: https://github.com/rails/rails/pull/12976
[6]: https://github.com/SamSaffron/fast_blank
