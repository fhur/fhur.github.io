---
layout: post
title:  "Writing a template language with clojure"
date:   2015-06-12 13:13:08
categories: mustache parsing templates
---

[Mustache](https://github.com/mustache/mustache) is a very sweet
template system inspired by ctemplate and el. Ever wondered how mustache
is implemented or how to write your own template system? this post will
teach you the basics.

NOTE: although this post discusses abstract syntax trees and lexical
analysis, don't be intimidated, no experience other than a basic knowledge of
Clojure and regular expressions is needed to read, understand and be able to
implement your own template language.

A simple example of how our templates will look like:
{% highlight clojure %}
{% raw %}
;; First we define the template
(def template
"<p> Hello {{name}}, your friends are: </p>
 <ul>
 {{#friends}}
   <li>{{first-name}} {{last-name}} </li>
 {{/friends}}
 </ul>")

;; Now we render the template
(render template
        {:name "Bob"
         :friends [{:first-name "Bill" :last-name "Gates"
                    :first-name "Chuck" :last-name "Cheese"}]})

;; Output should be something like:
"<p> Hello Bob, your friends are: </p>
 <ul>
   <li>Bill Gates</li>
   <li>Chuck Cheese</li>
 </ul>"
{% endraw %}
{% endhighlight %}

# Get the code

This blog post is based on [gabo](https://github.com/fhur/gabo), I
suggest you take a look at the README, clone the project or at least
play with it a little using the repl. Code is also available in
[clojars](http://clojars.org/gabo).


# Introduction

Rendering templates is a 3 step process:

##### **Step 1: Lexical Analysis**
The purpose of the *lexical analysis* step is to convert a template
string into a list of tokens.

##### **Step 2: Parsing**
The second step, *parsing*, is in charge of converting the list of
tokens produced by the lexical analysis step into an [abstract syntax
tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree).

##### **Step 3: Rendering**
The final step takes as input an abstract syntax tree and some input and
renders the given input using the template as structure.

Now that we have a high level understanding of the steps required to
compile and render templates let's get into detail.

# Step 1: Lexical Analysis

The main purpose of the *lexical analysis* step is to implement the
`tokenize` function which takes a template as input and converts it into
a list of tokens. A token is a very simple data structure (usually a
tuple) that splits the template into meaningful lexical elements. A
typical programming language will have string tokens, reserved keyword
tokens, number tokens, etc.

Let's see with an example how the `tokenize` function works.

Example 1: tokenizing a simple template
{% highlight clojure %}
{% raw %}
user=> (use 'gabo.lexer)
user=> (tokenize "Hi {{name}}")
[[:literal "Hi "] [:symbol "name"]]
{% endraw %}
{% endhighlight %}

Example 2: tokenizing a more complex template
{% highlight clojure %}
{% raw %}
user=> (tokenize "{{#people}}Name: {{name}}, age: {{age}}{{/people}}")
[[:iter-init "people" :default]
 [:literal "Name: "]
 [:symbol "name"]
 [:literal ", age: "]
 [:symbol "age"]
 [:iter-end "people"]]
{% endraw %}
{% endhighlight %}

*Note*: for now let us ignore the `:default` keyword present in the first
token, we will get to that later.

#### Tokens and their syntax:

All tokens are vectors of 2 or more elements. The first one is called
the token identifier, the second one corresponds to the token's value.

There are 4 types of tokens:

- `:symbol`: Symbols use the following syntax {%raw%}`{{something}}`{%endraw%}.
- `:iter-init`: Mark the beginning of an iteration, they have the
following syntax: {%raw%}`{{#something}}`{%endraw%}.
- `:iter-end`: they mark the termination of an iteration as started by
an :iter-init token. They look like {%raw%}`{{/something}}`{%endraw%}
- `:literal`: Literals match anything that is not a `symbol`, `iter-init` or
`iter-end`

#### Tokenization
Tokenization is the process of converting a string into a list of
tokens, the tokenization procedure is implemented using two functions:
`tokenize` and `tokenize-chunk`:

##### The tokenize function
{% highlight clojure %}
{% raw %}
(defn tokenize
  [template]
  (loop [remaining template
         tokens []]
    (if (empty? remaining)
      tokens
      (let [[str-match token] (tokenize-chunk remaining)]
        (recur (.substring remaining (count str-match))
               (conj tokens token))))))
{% endraw %}
{% endhighlight %}

The `tokenize` function is a very simple one, it takes as input a
template string and loops through the template until it reaches the end.
On every iteration the `tokenize-chunk` function is called which tries
to match the remaining template to a token, when a match is found, the
`tokens` vector will `conj`oined with the new token and the remaining
template will be `substring`ed by the matched string.

As you probably guessed, the `tokenize-chunk` is the function that is
doing the actual heavy work of matching strings to tokens.

{% highlight clojure %}
{% raw %}
(defn tokenize-chunk
  "Given a template substring returns a tuple of [match token]
  where match is the exact string that matched and token is
  the token that corresponds to the regex match.
  Example: (tokenize-chunk '{{bar}} foo') will return
  ['{{bar}}' [:symbol 'bar']]"
  [template]
  (when-match template
    ;; match symbols e.g. {{foo}}
    [sym #"\A\{\{\s*([\w-\.]+)\s*\}\}"]
      [ (first sym) [:symbol (last sym) ] ]

    ;; match iter-end e.g. {{/foo}}
    [iter-end #"\A\{\{/\s*(\w+)\s*\}\}"]
      [ (first iter-end) [:iter-end (last iter-end) ]]

    ;; match iter-init with no args e.g. {{#foo}}
    [iter-init #"\A\{\{#\s*(\w+)\s*\}\}"]
      [ (first iter-init) [:iter-init (last iter-init) :default ]]

    ;; match iter-init with args e.g. {{#foo 'separator'}}
    [iter-init #"\A\{\{#\s*(\w+)\s+'([\s\S]*?)'\s*\}\}"]
      [(first iter-init) (cons :iter-init (rest iter-init))]

    ;; match literals
    [literal #"\A([\s\S][\s\S]*?)\{\{"]
      [(last literal) [:literal (last literal)]]

    ;; more literals
    [literal #"\A[\s\S]*"]
      [literal [:literal literal]]))
{% endraw %}
{% endhighlight %}

#### `when-match`, what is that?
You probably noticed the use of the `when-match` macro. The implementation
for the `when-match` macro can be found
[here](https://github.com/fhur/gabo/blob/master/src/gabo/util.clj).

##### Example usage of `when-match`:

{% highlight clojure %}
{% raw %}
user=> (when-match "a long string"
  #_=>   [match #"long"] (keyword match)
  #_=>   [wont-match #"string"] (keyword wont-match))
:long
{% endraw %}
{% endhighlight %}

`when-match` takes a string as its first argument followed by n pairs of forms.
The macro will try to match the regular expression in the first form. If a
match is found then the symbol preceding the regular expression will be
`let` and the second form will be evaluated. Regular expression matches
use `re-find` under the hood.

If there is no match then the same will be attempted with the next pair
of forms. I suggest you play around with `when-match` in your `REPL` to
get a solid understanding of how it works.

Going back to the `tokenize-chunk` function: Implementation is very
simple once you understand `when-match`, each regex matches to one type
of token, when a token is matched a pair of `[string token]` will be
returned where `string` is the actual string that was matched and `token`
is the corresponding token constructed from the matched string.

##### Example: `tokenize-chunk` at work.

{% highlight clojure %}
{% raw %}
user=> (tokenize-chunk "Hi {{name}}, how are you?")
["Hi " [:literal "Hi "]]
user=> (tokenize-chunk "{{name}}, how are you?")
["{{name}}" [:symbol "name"]]
user=> (tokenize-chunk ", how are you?")
[", how are you?" [:literal ", how are you?"]]
{% endraw %}
{% endhighlight %}

Notice how this example tokenizes the {% raw %}`"Hi {{name}}, how are
you?"`{% endraw %} by matching chunk by chunk to a token.

We are done with the *lexical analysis* step. Let's go now to the *parsing*
step.

# Step 2: Parsing

Let us recall that the purpose of this step is to produce an *abstract
syntax tree* or AST. Think of an AST as a tree like representation of our
template which we can later use to actually render templates.

Consider a template which renders a country and every city in the
country. We will first tokenize it and then manually convert the tokens
into an AST.

{% highlight clojure %}
{% raw %}
(def template "Countries with their cities:
{{#countries '\n'}}
Country: {{name}}
Cities:
{{#cities '\n'}}
 * {{name}}
{{/cities}}
{{/countries}}")
{% endraw %}
{% endhighlight %}

Tokenization is quite simple using our `tokenize` function
{% highlight clojure %}
{% raw %}
user=> (tokenize template)
[[:literal "Countries with their cities:\n"]
 [:iter-init "countries" "\n"]
 [:literal "\nCountry: "]
 [:symbol "name"]
 [:literal "\nCities:\n"]
 [:iter-init "cities" "\n"]
 [:literal " * "]
 [:symbol "name"]
 [:iter-end "cities"]
 [:literal "\n"]
 [:iter-end "countries"]]
{% endraw %}
{% endhighlight %}

We will now convert those tokens into a tree. The conversion will use
these rules:

* `:literal` or `:symbol` tokens will be left untouched
* `:iter-init` and `:iter-end` tokens will be grouped into an `:iter`
node and all tokens between the init/end pair will be parsed
recursively.

##### Example of an AST as produced by the `parse` function
{% highlight clojure %}
{% raw %}
user=> (parse template)
[[:literal "Countries with their cities:\n"]
 [:iter "countries" "\n" [[:literal "\nCountry: "]
                          [:symbol "name"]
                          [:literal "\nCities:\n"]
                          [:iter "cities" "\n" [[:literal "\n * "]
                                                [:symbol "name"]
                                                [:literal "\n"]]]
                          [:literal "\n"]]]]
{% endraw %}
{% endhighlight %}

Let's now take a look at the parse function:

{% highlight clojure %}
{% raw %}
(defn parse
  [string]
  (-> (tokenize string)
       build-ast))
{% endraw %}
{% endhighlight %}

It doesn't do much, let's take a look at the `build-ast` which takes a
list of tokens as input and outputs an AST.

{% highlight clojure %}
{% raw %}
(defn- build-ast
  "Builds an abstract syntax tree given a list of tokens as produced by tokenize"
  [tokens]
  (loop [tokens tokens
         ast []]
    (if (empty? tokens)
      ast
      (let [token (first tokens)]
        (cond
          (or (is-literal token) (is-symbol token))
            (recur (rest tokens)
                   (conj ast token))
          (is-iter-init token)
            (let [sub-list (find-iter-sub-list tokens)
                  [_ token-val separator] token]
              (recur (drop (+ 2 (count sub-list)) tokens)
                     (conj ast [:iter token-val
                                      separator
                                      (build-ast sub-list)])))
          :else
            (throw (unexpected-token-exception token)))))))
{% endraw %}
{% endhighlight %}

The `build-ast` function is a bit complex, but let's break it down into
a series of steps:

* It `loop`s through all the tokens; the `ast` which will be the
result of the function.
* The `if` in line 6 simply asserts that when there are no tokens left,
we are done so just return the `ast`.
* If there are tokens left then line 8 `let`s `token` to the first token
in the iteration.
* The `cond` has 2 cases and an `else`
  * Case 1: the "simple" case, if the token is a literal or symbol
  simply add token to the `ast`.
  * Case 2: the "hard" case is evaled when an `:iter-init` token is
  found.   
  The `find-iter-sub-list` function will return all tokens inside
  the `:iter-init` and `:iter-end` pair.
  The `recur` call will drop all tokens found in the `sub-list` + 2
  which corresponds to the `:iter-init` and `:iter-end` tokens.
  The `ast` is then `conj`oined to a recursive call of the `build-ast`
  procedure over the `sub-list` of tokens.
  * `else` case: if something other than an `:iter-init`, `:symbol` or
  `:literal` is found, then throw an exception because we found an
  unexpected token.

This finishes our explanation of the parsing step. I have tried to write
the simplest possible parser, hopefully you think so too. A more complex
one would, for example, include better error detection mechanisms.

# Step 3: Rendering

We have arrived to the final step in our template rendering system. This
system builds upon steps 1 and 2 and takes an AST and a context as
arguments and returns a `render`ed template. A context is simply a map
where every key is a `keyword` and `vals` are numbers, strings or
collections of contexts.

{% highlight clojure %}
{% raw %}
;; The {{.}} refers to the item that is being iterated.
user=> (def tokens (parse "{{#numbers}}Number:{{.}}{{/numbers}}"))
user=> tokens
user=> [[:iter "numbers" :default [[:literal "Number:"]
                                   [:symbol "."]]]]
user=> (eval-tree tokens {:numbers [1 2 3 4 5]})
"Number:1,Number:2,Number:3,Number:4,Number:5"
{% endraw %}
{% endhighlight %}

Let us now look at the implementation of `eval-tree`:
{% highlight clojure %}
{% raw %}
(defn eval-tree
  "Evaluates a compiled template as a tree with the given context"
  [tree ctx]
  (cond (is-literal tree)
          (second tree)
        (is-symbol tree)
          (let [[_ token-val] tree]
            (if (= token-val ".")
              (str ctx)
              (get ctx (keyword token-val) "")))
        (is-iter tree)
          (let [[_ token-val separator sub-tree] tree
                coll (get ctx (keyword token-val) [])]
            (->> (map #(eval-tree sub-tree %) coll)
                 (interpose (if (= :default separator) "," separator))
                 (apply str)))
        ;; Only executed once: the first time eval-tree is called, no subsequent
        ;; recursive call will go through this branch.
        (coll? tree)
          (apply str (map #(eval-tree % ctx) tree))))
{% endraw %}
{% endhighlight %}

`eval-tree` runs through each node in the AST recursively as follows:

* If a `:literal` token is found, return the token's value.
* If a `:symbol` is found then check if its value is `"."`
  * Case `token-val` is `"."`: return the complete context as a string.
  * Case `token-val` is not `"."`: look for `token-val` as a keyword
  inside the context map and return that.
* If an `:iter` tree is found then:
  * `let` `coll` to to the collection inside the context that matches
  the `:iter`'s `token-val`.
  * Recursively call `eval-tree` on every node in the `sub-tree`.
  * Interpose the previous result with the separator which is by default
  `','`.
  * Finally concatenate all "sub-templates" and separators.
* The final branch `(coll? tree)` is actually only evaled once, the first
time that `eval-tree` is called as ASTs produced by `parse` are not
actually trees, they are a list of trees. This step is equivalent to the
previous branch *sans* interposing with separators.

# Wrapping things up

Now that we have implemented `tokenize`, `parse` and `eval-tree`. The
`render` function is very simple:

{% highlight clojure %}
{% raw %}
(defn render
  "Compiles and evaluates the template with the given context"
  [template ctx]
  (eval-tree (parse template) ctx))
{% endraw %}
{% endhighlight %}

#### Hooray!
I hope you now have a good understanding on how to write your own
template system. If you have any questions, suggestions or just want to
chat, you can e-mail me at `fernandohur` at `gmail.com`.

Criticism is always welcome but go easy on me, it's my first post ;)
