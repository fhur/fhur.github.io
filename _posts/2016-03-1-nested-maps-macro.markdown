---
layout: post
title:  "A simple macro to get deeply nested values in maps"
date:   2016-03-1 13:13:08
categories: macro maps
---

I hate it when I have to write a lot of code to do simple things. Recently I
had to write some code to get a value in a deeply nested associative structure.
Consider the following example:

{% highlight clojure %}
{% raw %}
(def complex-object
  {:foo {:bar {:baz {:qux "deep inside"}}}})
{% endraw %}
{% endhighlight %}

The immediately obvious solution is a pain:

{% highlight clojure %}
{% raw %}
(def complex-object
  {:foo {:bar {:baz {:qux "deep inside"}}}})

(get (get (get (get complex-object :foo) :bar) :baz) :qux)
;; => deep inside
{% endraw %}
{% endhighlight %}

However a smart clojurian quickly notices that he can use the awesome `->`
macro to simplify things:

{% highlight clojure %}
{% raw %}
(def complex-object
  {:foo {:bar {:baz {:qux "deep inside"}}}})

(-> complex-object
    (get :foo)
    (get :bar)
    (get :baz)
    (get :qux))
;; => deep inside
{% endraw %}
{% endhighlight %}

But that's still way to long. Can we do better? Sure we can:

{% highlight clojure %}
{% raw %}
(def complex-object
  {:foo {:bar {:baz {:qux "deep inside"}}}})

(get-in complex-object [:foo :bar :baz :qux])
;; => deep inside
{% endraw %}
{% endhighlight %}

Now this is much better. Still I though I would write my own macro
just for the fun of it, so here you go:

{% highlight clojure %}
{% raw %}
(defmacro inside
  [sym]
  (let [args (clojure.string/split (str sym) #"\.")
        head (symbol (first args))
        ks (map keyword (rest args))]
    (concat `(-> ~head) (map (fn [k] `(get ~k)) ks))))

;; Now you can do awesome things like:
(inside x.foo.bar.baz.qux)
;; => "deep inside"
{% endraw %}
{% endhighlight %}

Voila! A simple macro to make you code a bit nicer and avoid boilerplate.
