---
layout: post
title: "Ruby 2.1: objspace.so"
---

ObjectSpace in ruby contains many useful heap debugging utilities.

Since 1.9 ruby has included `objspace.so` which adds even more methods to the ObjectSpace module:

``` ruby
ObjectSpace.each_object{ |o| ... }
ObjectSpace.count_objects #=> {:TOTAL=>55298, :FREE=>10289, :T_OBJECT=>3371, ...}

require 'objspace'
ObjectSpace.memsize_of(o) #=> 0 /* additional bytes allocated by object */
ObjectSpace.count_tdata_objects #=> {Encoding=>100, Time=>87, RubyVM::Env=>17, ...}
ObjectSpace.count_nodes #=> {:NODE_SCOPE=>2, :NODE_BLOCK=>688, :NODE_IF=>9, ...}
ObjectSpace.reachable_objects_from(o) #=> [referenced, objects, ...]
ObjectSpace.reachable_objects_from_root #=> {"symbols"=>..., "global_tbl"=>...}
```

In 2.1, we've added a two big new features: an allocation tracer and a heap dumper.

### Allocation Tracing

Tracking down memory growth and object reference leaks is tricky when you don't know where the objects are coming from.

With 2.1, you can enable allocation tracing to collect metadata about every new object:

``` ruby
require 'objspace'
ObjectSpace.trace_object_allocations_start

class MyApp
  def perform
    "foobar"
  end
end

o = MyApp.new.perform
ObjectSpace.allocation_sourcefile(o) #=> "example.rb"
ObjectSpace.allocation_sourceline(o) #=> 6
ObjectSpace.allocation_generation(o) #=> 1
ObjectSpace.allocation_class_path(o) #=> "MyApp"
ObjectSpace.allocation_method_id(o)  #=> :perform
```

A block version of the tracer is [also available](http://ruby-doc.org/stdlib-2.1.0/libdoc/objspace/rdoc/ObjectSpace.html#method-c-trace_object_allocations).

Under the hood, this feature is built on `NEWOBJ` and `FREEOBJ` tracepoints included in 2.1. These events are only available from C, via `rb_tracepoint_new()`.

### Heap Dumping

To further help debug object reference leaks, you can dump an object (or the entire heap) for offline analysis.

``` ruby
require 'objspace'

# enable tracing for file/line/generation data in dumps
ObjectSpace.trace_object_allocations_start

# dump single object as json string
ObjectSpace.dump("abc".freeze) #=> "{...}"

# dump out all live objects to a json file
GC.start
ObjectSpace.dump_all(output: File.open('heap.json','w')))
```

Objects are serialized as simple json, and include all relevant details about the object, its source (if allocating tracing was enabled), and outbound references:

``` json
{
 "address":"0x007fe9232d5488",
 "type":"STRING",
 "class":"0x007fe923029658",
 "frozen":true,
 "embedded":true,
 "fstring":true,
 "bytesize":3,
 "value":"abc",
 "encoding":"UTF-8",
 "references":[],
 "file":"irb/workspace.rb",
 "line":86,
 "method":"eval",
 "generation":15,
 "flags":{"wb_protected":true}
}
```

The heap dump produced by `ObjectSpace.dump_all` can be processed by the tool of your choice. You might try a [json processor like jq](http://stedolan.github.io/jq/) or a [json database](http://www.rethinkdb.com/). Since the dump contains outbound references for each object, a full object graph can be re-created for deep analysis.

For example, here's a simple ruby/shell script to see which gems/libraries create the most long-lived objects of different types:

``` console
$ cat heap.json |
    ruby -rjson -ne ' puts JSON.parse($_).values_at("file","line","type").join(":") ' |
    sort        |
    uniq -c     |
    sort -n     |
    tail -4

26289 lib/active_support/dependencies.rb:184:NODE
29972 lib/active_support/dependencies.rb:184:DATA
43100 lib/psych/visitors/to_ruby.rb:324:STRING
47096 lib/active_support/dependencies.rb:184:STRING
```

If you have a ruby application that feels large or bloated, give these new ObjectSpace features a try. And if you end up writing a heap analysis tool [or visualization](http://arborjs.org) for these json files, do let me know [on twitter](https://twitter.com/tmm1).
