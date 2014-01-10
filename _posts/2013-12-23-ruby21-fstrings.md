---
layout: post
title: "Ruby 2.1: Frozen String Literals"
tagline: '30% fewer long-lived strings + "".freeze optimization'
---

In Ruby 2.1, `"str".freeze` is optimized by the compiler to return a single shared frozen strings on every invocation. An alternative `"str"f` syntax was implemented initially, but later reverted.

Although the external scope of this feature is limited, internally it is used extensively to de-duplicate strings in the VM. Previously, every `def method_missing()`, the symbol `:method_missing` and any literal `"method_missing"` strings in the code-base would all create their own String objects. With Ruby 2.1, only one string is [created and shared](https://bugs.ruby-lang.org/issues/9171). Since many strings are commonly re-used in any given code base, this easily adds up. In fact, large applications can expect up to [30% fewer long-lived strings](https://bugs.ruby-lang.org/issues/9159) on their heaps in 2.1.

For 2.2, there are plans to [expose this feature via a new `String#f`](https://bugs.ruby-lang.org/issues/9229). There's also a proposal for a [magic `immutable: string` comment](https://bugs.ruby-lang.org/issues/9278) to make frozen strings default for a given file.
