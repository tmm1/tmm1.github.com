---
layout: post
title: "Ruby 2.1: Process.clock_gettime()"
---

Cpu vs idle time is one of the first things I always look at when profiling rails requests.

Cpu time consists of number crunching, template rendering, method invocation and any other time spent executing instructions on the CPU. Idle time is everything else- generally this is time spent waiting on disk or network I/O, and can be highly variable depending on network conditions, remote server load and other factors.

In the past I've used ruby-prof's `RubyProf::Measure::ProcessTime.measure` to measure cpu time, but with Ruby 2.1 we have a `clock_gettime(3)` wrapper built-in!

``` ruby
def time
  real, cpu = Time.now, Process.clock_gettime(Process::CLOCK_PROCESS_CPUTIME_ID)
  yield
  real = Time.now-real
  cpu = Process.clock_gettime(Process::CLOCK_PROCESS_CPUTIME_ID) - cpu
  { real: real, cpu: cpu, idle: real-cpu }
end
```

``` irb
>> time{ sleep 1 }                           # all idle time
=> {:real=>1.000452, :cpu=>0.00041599999999997195, :idle=>1.000036}

>> time{ 10000.times{2**65536} }             # all cpu time
=> {:real=>0.21192, :cpu=>0.211714, :idle=>0.00020599999999998397}

>> time{ open('http://google.com').read }    # mixed, mostly idle
=> {:real=>0.342832, :cpu=>0.05224400000000001, :idle=>0.290588}
```

The method also takes an optional second argument `unit`, which can be `:millisecond`, `:microsecond`, `:nanosecond`, or a `:float_` variant thereof. See [the documentation](http://ruby-doc.org/core-2.1.0/Process.html#method-c-clock_gettime) for more details.
