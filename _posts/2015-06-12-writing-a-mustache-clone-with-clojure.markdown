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
{{/friends}}")

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

As you can see from the example, the `tokenize` takes as input a
mustache template and outputs a list of tokens. In this blog post we
will only cover the implementation of the `:literal`, `:symbol`,
`iter-init` and `:iter-end` tokens which will be defined as follows:

- `:symbol` token: A `:symbol` token matches the
`{% raw %}{{foo}}{% endraw %}` as a `[:symbol "var-name"]` pair.
- `:iter-init` token: matches the `{% raw %}{{#loop}}{% endraw %}`
syntax as a `[:iter-init "loop"]` pair.
- `:iter-end` token: matches the `{% raw %}{{/loop}}{% endraw %}`
syntax as a `[:iter-end "loop"]` pair.
- `:literal` token: matches anything that is not matched by the
previously defined tokens as a `[:literal "text stuff"]`

A full tokenization example:

{% highlight clojure %}
{% raw %}

(def template
"<h1> this is a literal </h1>
 <p> Hi {{name}}, </p>
 Your friends are:
 {{#friends}}
 <p> Name: {{name}} </p>
 {{/friends}}")

(tokenize template)
;; expected output
[[:literal "<h1> this is a literal </h1>\n<p> Hi "]
 [:symbol "name"]
 [:literal ", </p>\nYour friends are:\n"]
 [:iter-init "friends" ]
 [:literal "\n<p> Name: "]
 [:symbol "name"]
 [:literal " </p>\n"]
 [:iter-end "friends"]]

{% endraw %}
{% endhighlight %}

This example should give you a reasonable understanding of how
to tokenize mustache templates. The purpose of the tokenization function
is to convert a mustache string into a datastructure which will be much
easier to work with. Now let's see some code!

Let's first take a look into the tokenize function.
{% highlight clojure %}
{% raw %}

(defn tokenize
  [code]
  (loop [code code
         tokens []]
    (if (empty? code)
      tokens
      (let [[str-match token] (tokenize-chunk code)]
        (recur (.substring code (count str-match))
               (conj tokens token))))))

{% endraw %}
{% endhighlight %}

The tokenize function is very simple, it just loops through a `code`
string and calls the `tokenize-chunk` function on every iteration of the
loop unless the `code` string is empty which means the tokenization
procedure finished.

The `tokenize-chunk` function returns a tuple, the first item being the
actual string that was matched and the second item being the token e.g.
`[["{{foo}}"] [:symbol "foo"]]`

{% highlight clojure %}
{% raw %}
(defn tokenize-chunk
  "Given a string of code return a tuple of [match token]
  where match is the exact string that matched and token is
  the token that corresponds to the regex match.
  Example: (tokenize-chunk '{{bar}} foo') will return
  ['{{bar}}' [:symbol 'bar']]"
  [code]
  (when-match code
    [sym #"\A\{\{\s*([\w-\.]+)\s*\}\}"] [ (first sym) [:symbol (last sym) ] ]
    [iter-end #"\A\{\{/\s*(\w+)\s*\}\}"] [ (first iter-end) [:iter-end (last iter-end) ]]
    [iter-init #"\A\{\{#\s*(\w+)\s*\}\}"] [ (first iter-init) [:iter-init (last iter-init) :default ]]
    [iter-init #"\A\{\{#\s*(\w+)\s+(\S+?)\s*\}\}"] [(first iter-init) (cons :iter-init (rest iter-init))]
    [literal #"\A([\s\S][\s\S]*?)\{\{"] [(last literal) [:literal (last literal)]]
    [literal #"\A[\s\S]*"] [literal [:literal literal]]))

{% endraw %}
{% endhighlight %}

Before I talk about the `tokenize-chunk` function I must first explain
what the `when-match` macro does.

{% highlight clojure %}
{% raw %}
(defmacro when-match
  "Syntax:
  (when-match string
    [sym1 regex1] form1
     ...
    [symN regexN] formN)

  Usage: matches string to every regex, if there is a match then
  the form will be evaluated and returned.
  "
  [string & forms]
  (cons `cond
        (loop [form-pairs (partition 2 forms)
               result []]
          (if (empty? form-pairs)
            result
            (let [[[sym regex] body] (first form-pairs)]
              (recur (rest form-pairs)
                     (conj result
                           `(re-find ~regex ~string)
                           `(let [~sym (re-find ~regex ~string)]
                              ~body))))))))

{% endraw %}
{% endhighlight %}

The `when-macro` is just syntax sugar: it takes a string as its first
argument and a list of `[sym regex] action`. The `when-match` will
attempt to match the string if every regex in order, if a match is found
then the `sym` will be `let` to the match and `action` will be executed.
the `action`. I'm actually new to macros so this might not reflect best
practices, all suggestions are welcome.

Going back to our `tokenize-chunk` function, you can see we define a
list of regular expressions and the variables that are `let` when the
string matches. The matched string is then converted to a tuple where
the first item is the string that matched and the second item is the
resulting token.

To get a better understanding of how `when-match`, `tokenize-chunk` and
`tokenize` works I reccomend you open up a `REPL` and paste `when-match`
followed by `tokenize-chunk` and `tokenize` and just play a little.

Needless to say, the `when-match` function is not necessary, I just
prevents the `tokenize-chunk` function from becoming too verbose.

#### Building the parse tree

This is the last step in what we can call the "template compilation"
process. The input of this step is a list of tokens and the output will
be a tree of nodes which describe the template structure and which will
later be fed to the rendering engine.

Let us introduce what this step is all about with an example. We will
build upon the last tokenization example and use the tokens to create a
parse tree.

{% highlight clojure %}
{% raw %}

;; This are the tokens from our last example
(def tokens [[:literal "<h1> this is a literal </h1>\n<p> Hi "]
             [:symbol "name"]
             [:literal ", </p>\nYour friends are:\n"]
             [:iter-init "friends" ]
             [:literal "\n<p> Name: "]
             [:symbol "name"]
             [:literal " </p>\n"]
             [:iter-end "friends"]])

(build-parse-tree tokens)
;; Output will be
[[:literal "<h1> this is a literal </h1>\n<p> Hi "]
 [:symbol "name"]
 [:literal ", </p>\nYour friends are:\n"]
 [:iter "friends" :default
   [[:literal "\n<p> Name: "]
    [:symbol "name"]
    [:literal " </p>\n"]]]]

{% endraw %}
{% endhighlight %}

As you might have probably noticed already, the output of the
`build-parse-tree` function is not a list of tokens but a tree.

#### Rendering



