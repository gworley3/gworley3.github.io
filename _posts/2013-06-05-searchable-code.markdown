---
layout: post
category: programming
title:  Searchable Code
date:   2013-06-05
listed: false
permalink: /2013/06/05/searchable-code.html
published: true
---

*This post originally appeared on my old blog. Copied over a slightly revised on 2014-07-12 (1405197000).*

Code should be searchable. When I'm working on something, I want to be able to search through the entire code base for a symbol and reliably find all places where that symbol is used and, perhaps most importantly, find where that symbol is defined. For example, take this bit of Ruby code I've seen:

```ruby
class LogRecord
  ...
  def uuid
    try(uid) || generate_uuid
  end
  ...
end
```

In this case, `uid` is an instance variable on `LogRecord` and `generate_uuid` is a class method. Now, you would expect to be able to search through the code and find somewhere that `uid` is at least defined if not set. No such luck. But if you stare long enough you might notice this code in `LogRecord` and get suspicious:

```ruby
fields.keys.each do |field|
  class_eval <<-END
    def #{field}
      fields[field]
    end
  END
end
```

What's happening here is that we are dynamically creating a method for each key in the `fields` hash to allow accessing it via dot notation by name on a `LogRecord` instance. Unfortunately there was no way for me to find this chunk of code by searching for any string in the first chunk of code.

If you're lucky, you'll find something like the following in the codebase that will allow you to connect `uid` and the above code chunk.

```ruby
  ...
  fields = {'uid' => '12345', ...}
  ...
```

Then, once you find `uid`, you can try searching for fields and then hopefully, after some indirection, find what you want. But, to make matters worse, suppose that `fields` isn't defined statically, but populated dynamically at runtime from a value set somewhere outside the codebase. Now you have no way to connect `uid` to its definition. Boo!

The problem isn't with metaprogramming or Ruby, but in the way that they combine. Since Ruby is object-oriented, it's natural to want to define methods on objects using metaprogramming, and it's common Ruby style to use metaprogramming to create methods for individual instance values rather than generic `get_field` or `set_field` methods. And while Ruby style certainly leads to code that looks nice and is easy to work with once you understand it sufficiently, it has the unintended consequence of hurting searchability in this context because there is no symbol that ties the code chunks together directly.

Lest you get the wrong idea, I'm not here to pick on Ruby: it's just the language I'm working in at the moment where I sometimes have problems searching code and I have a ready example. These kinds of searchability problems can pop up in any language and take many different forms, but they are united by a common feature: searching for a symbol does not lead you to understanding what a symbol represents or does.

I know of no magic prescription to improve searchability in your code, but I will give you a list of things to know about searchability that I can think of now.

- Every symbol in every line of code should be searchable (language symbols like `if` or `function` excluded)
- A symbol is searchable if searching for that symbol in your codebase finds where that symbol is defined
- Any indirection in searchability impedes understanding (e.g. the above example demonstrates indirection)
- Speed matters: the faster I can search for something in a codebase the more time I can spend doing stuff I care about
- Too many symbols that are substrings of other symbols makes searching slow
- If a symbol comes from a library and that library's source is not in your codebase, make sure it's clear what library the symbol comes from (I'm looking at you, programmer who merges library namespaces into your namespaces)

There is, of course, a lot more to writing good code than writing searchable code, and sometimes you may have to trade off searchability in order to write the best code you can. But, in my experience, searchability is often undervalued, so we coders have much to gain if we start to give it the same kind of consideration we give issues like encapsulation, code reuse, and clarity.
