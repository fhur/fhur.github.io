---
layout: post
title:  "A simple macro to simulate clojure interpolation"
date:   2016-03-1 12:13:08
categories: macro clojure interpolation ruby
---

# Ruby interpolation

One features that Ruby has that makes life easier is string interpolation.
With string interpolation you can make your code easier on the eyes when there
is a lot of string concatenation to be done:

{% highlight ruby %}
{% raw %}
# without interpolation
"Hi, " + name + " " + last_name + ", you last logged in at " + time + "."
# with interpolation
"Hi, #{name} #{last_name}, you last logged in at #{time}."
{% endraw %}
{% endhighlight %}

which in clojure would look something like:

{% highlight clojure %}
{% raw %}
(str "Hi, " name " " last-name ", you last logged in at " time ".")
{% endraw %}
{% endhighlight %}

# Ruby style string interpolation
Clojure unfortunately does not have string interpolation, but thanks to macros
we can fix this.

We will implement a macro that looks like this:

{% highlight clojure %}
{% raw %}
(itp "Hi, #{name} #{last-name}, you last logged in at #{time}.")
;; => "Hi, George Washington, you last logged in at 10:00am."
{% endraw %}
{% endhighlight %}

And expands to:

{% highlight clojure %}
{% raw %}
(str "Hi, " name " " last-name ", you last logged in at " time ".")
{% endraw %}
{% endhighlight %}

### Limitations:
To keep things simple, you can only enter one symbol inside `#{}`, you can't
enter complex expressions like `#{(conj [1 2 3 4] 6)}`.
Also we won't be doing escaping or other fancy things, so don't try to do
`(itp "foo #{ {bar})"` which will confuse our simple implementation.

## Step one: parse the string
The first thing we need to do is to implement a function that given a string
with interpolations, return an array of strings and symbols.

1. Take input string and iterate through characters until you reach the end
   of the string or you reach an `#{interpolation}`.
2. Split the string into two parts: the first part is a 'raw' string, the second
   part will be an interpolation. Parse the interpolation and extract the
   inner symbol.

{% highlight clojure %}
{% raw %}
(defn parse-interpolation
  [string]
  ;; iterate over the string, store the result in the parts
  (loop [string string
         parts []]
    (let [[[_ str-part sym-part]] (re-seq #"^(.*?)#\{(.*?)\}" string)]
      (if (and str-part sym-part)
        ;; the 3 here accounts for the '#', '{' and '}' which were stripped
        ;; of by the re-seq above.
        (recur (.substring string (+ (count str-part) 3
                                     (count sym-part)))
               (conj parts str-part (symbol sym-part)))
        (conj parts string)))))

{% endraw %}
{% endhighlight %}

## Step two: build the macro
Having defined the `parse-interpolation` function in the previous step, it is
now trivial to define our interpolation macro as follows:

{% highlight clojure %}
{% raw %}
(defmacro itp [string]
  (cons 'str (parse-interpolation string)))

;; Usage:
(let [name "Billy"
      last-name "The Kid"
      time "10:00pm"]
  (itp "Hi #{name} #{last-name}, you last logged in at #{time}."))
;; => "Hi Billy The Kid, you last logged in at 10:00pm."

{% endraw %}
{% endhighlight %}

## Interpolation!

We now have an interpolation macro that will make our string concatenations
more readable.
