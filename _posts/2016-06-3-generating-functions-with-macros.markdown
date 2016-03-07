---
layout: post
title:  "A simple macro to simulate ruby string interpolation"
date:   2016-06-3 12:13:08
categories: macro clojure
---

# How to generate functions with macros

This blog post will show you how to use macros to write functions with similar
behavior. This general method will greatly reduce the amount of code that
you need to type and will keep your codebase clean and easy to test.

Let's assume for a moment that you are building a system where you have users.
A user has several fields including `name`, `email`, `phone-number` and
`gender`. For the purposes of this article we will model a user as a simple map.

#### The user data structure.
{% highlight clojure %}
{% raw %}
{:name "Bob"
 :email "bob@mailinator.com"
 :phone-number "+56 123456789"
 :gender :male}
{% endraw %}
{% endhighlight %}

## Requirements

At some point in time, your boss comes into your office and says:
"Jim, we need to be able to find users in the database by name, email, phone
number and gender and we need it now!"

You are an awesome clojure developer so you quickly come up with a straight
forward solution:

{% highlight clojure %}
{% raw %}
(defn find-by-id [id]
  (->> (build-query "SELECT * FROM users WHERE id=%s" id) ;; creates an SQL query
       (run-query) ;; runs the query against the database
       (rows->maps))) ;; Iterate over the cursor and parse the rows to maps.

(defn find-by-email [email]
  (->> (build-query "SELECT * FROM users WHERE id=%s" email) ;; creates an SQL query
       (run-query) ;; runs the query against the database
       (rows->maps))) ;; Iterate over the cursor and parse the rows to maps.

;; ...
{% endraw %}
{% endhighlight %}

At this point you realize that you are repeating yourself and since you are an
awesome developer you start to think how you can do this better.

## Keeping it DRY with functions

You immediately realize: well what if I had a function `find-by-field` which
finds a user in the database given a field and a value for that field.

{% highlight clojure %}
{% raw %}
(defn find-by-field
  [field value]
  (->> (build-query "SELECT * FROM users WHERE %s=%s" field value) ;; creates an SQL query
       (run-query) ;; runs the query against the database
       (rows->maps))) ;; Iterate over the cursor and parse the rows to maps.

(defn find-by-email [email]
  "Finds and returns a list of users in the database with the given email"
  (find-by-email "email" email)

(defn find-by-gender [gender]
  "Finds and returns a list of users in the database with the given gender"
  (find-by-field "gender" gender))

(defn find-by-phone-number [phone-number]
  "Finds and returns a list of users in the database with the given phone-number."
  (find-by-field "phone_number" phone-number))

(defn find-by-name [name]
  "Finds and returns a list of users in the database with the given name."
  (find-by-field "name" name))
{% endraw %}
{% endhighlight %}

You look at your work and think, damn, that's a good simple, clean solution.
But the question remains, can we do better? Yes we can! Looking at your code
you quickly realize that the `find-by-{field}` functions can be re-writter
with macros which will remove all the needless boiler plate and code repetition.

## It's macro time!

As always when describing macros I like to first see how the macro invocation
will look like as well as the expanded code and then it's usually easier to
implement the macro.

{% highlight clojure %}
{% raw %}
(defuser-finders :name :gender :phone-number :email)
;; Will expand to
(do (defn find-by-name [name] (find-by-value "name" name))
    (defn find-by-gender [gender] (find-by-gender "gender" gender))
    ;; ... etc ...
{% endraw %}
{% endhighlight %}

The cool think about this approach is that it actually defines good ol'
functions which can be composed and used as you would normally use
functions anywhere else in your awesome clojure codebase.

So without further delay, here is the code: A 20-liner can be used to
define as many finder functions as possible.
{% highlight clojure %}
{% raw %}
(defn- field->db-field-name
  "Given a keyword like :phone-number, returns the database equivalent
  e.g. 'phone_number' as a string."
  [field-kw]
  (.replaceAll (name field-kw) "-" "_"))

(defn- field->finder-name
  "Given a keyword like :phone-number, returns a finder function name
  as a symbol e.g. find-by-phone-number"
  [field-kw]
  (symbol (str "find-by-" (name field-kw))))

(defmacro defuser-finders
  [& fields]
  (cons 'do
        (map (fn [field]
               `(defn ~(field->finder-name field)
                  ~(str "Finds and returns a list of users in the database "
                        "with the given " (name field) ".")
                  [arg#]
                  (find-by-field ~(field->db-field-name field) arg#)))
                  fields)))
{% endraw %}
{% endhighlight %}

## Wrapping it up

So there you go, a simple extensible approach with 0 performance degradation
that will make your code clean and extremely easy to extend. The key points to
note about the macro approach:

1. Adding another finder function is as easy as adding another argument to the
   `defuser-finders` macro invocation
2. The generated code is as performant as hand written code. Since macro
   expansion happens before runtime, these functions will actuallty be compiled
   into bytecode.
3. The generated code comes with documentation so users of your code can
   `(doc find-by-user)`.

I hope you liked this post. As always critticism and comments are more than
welcome. Cheers and happy coding!
