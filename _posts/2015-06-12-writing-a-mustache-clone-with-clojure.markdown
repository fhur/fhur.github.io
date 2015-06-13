---
layout: post
title:  "Writing a {{Mustache}} implementation with Clojure"
date:   2015-06-12 13:13:08
categories: jekyll update
---

[Mustache](https://github.com/mustache/mustache) is a very sweet
template system inspired by ctemplate and el. This blog post will
teach you how to build a minimal mustache implementation using Clojure.

A simple example of how Mustache templates look like:
{% highlight clojure %}
{% raw %}
;; First we define the template
(def mustache-template
"Hello {{name}}, your friends are
{{#friends}}
{{first-name}} {{last-name}}
{{/friends}}"

;; Now we render the template
(render mustache-template
        {:name "Bob"
         :friends [{:first-name "Bill" :last-name "Gates"
                    :first-name "Chuck" :last-name "Cheese"}]})

;; Output should be something like:
;; "Hello Bob, your friends are
;;  Bill Gates
;;  Chuck Cheese"

{% endraw %}
{% endhighlight %}

# Structure

This post will be structured in two parts, the first part will explain
how mustache will be implemented and the second part will actually
show a working implementation.

This post is not meant to provide a feature-full implementation of the
full mustache spec, just the basics.

# Part 1: Theory

There are many ways to implement templating languages, this is just one
of them. The process of rendering a template will be split into three
steps: lexical analysis, building the parse tree and rendering the
template.

#### Lexical analysis

This objective of the lexical analysis phase is to produce a list of
tokens given a mustache template string. First I will show you an
example and then I will explain how the 'lexing' procedure works.

Example:

{% highlight clojure %}
{% raw %}
(tokenize "Hi {{name}} this is a mustache template ")
;; expected output
[[:lit "Hi "] [:sym "name"] [:lit "this is a mustache template"] ]

{% endraw %}
{% endhighlight %}


#### Building the parse tree



#### Rendering



