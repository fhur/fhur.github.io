---
layout: post
title:  "Composable regular expressions"
date:   2016-06-3 12:13:08
categories: data-structure clojure
---

# Implementing a Union Find Data Structure

When I first started learning clojure one of the topics that puzzled me the most was how to implement
immutable datastructures efficiently. In C/Java based languages most (if not all) operations are simply
reduced to moving pointers/references around. This is both efficient and straightforward most of the time.

{% highlight java %}
{% raw %}
/**
 * Example: adding an element to a Queue
 **/
public void push(T element) {
  QueueNode lastNode = this.getTail();
  QueueNode newNode = new QueueNode(element, null);
  lastNode.next = newNode;
}
{% endraw %}
{% endhighlight %}

However when your start thinking about immutable datastructures, some operations which are usually
trivial, become hard or at least non-intuitive. The purpose of this blog post is to implement a Union Find
datastructure and show you how this can be achieved using clojure.

## Union & Find

To begin, let's define what a UnionFind datastructure is. If you are already familiar with UnionFind,
feel free to skip this section. I chose UnionFind as an interesting data structure to implement
because it's simple yet most developers (including me) don't use it on a daily basis so implementing
it will not be completely trivial.

The union find (UF) datastructure exposes two operations `union` and `find`. It is used most notably in
Kruskal's Minimum Spanning Tree Algorithm. A UF is composed of a collection of sets; given an element
you can quickly determine to which set the element belongs to (`find`) and given two sets, you can
merge them (`union`). UF also typically exposes a `makeset` operation which given an element returns
a singleton set with that element in it.

To get a bit of intuition behind UF consider that you have a set of gangsters and a map indicating where
was the last time that they were seen in action (possibly in some lucrative venture like stealing cookies
from little girl scouts). Your superb crime fighting skills tell you that gangsters
that have been spoted close to each other are likely to belong to the same gang. You are asked to identify
groups of gangsters that have been spotted in the same area so that you can get an idea of which gang they
belong to. Assume you also know beforehand the number of gangs operating in the city.

This classic clustering problem is easy to solve using UF:
1. Form a graph with all the gangsters as vertices (a graph G, get it?)
2. Find the closest pair of gangsters `(a, b)` such that `find(a) != find(b)` and call `union(a,b)`.
   Add an adge in the graph between `a` and `b`.
3. Repeat n-k times until there are exactly k gangs.

## SHOW ME THE CODEZ!

Now that we have a good intuition on why UF is an important and useful datastructure, let's see
how this would look like in Clojure. Let's being by describing the protocol:

{% highlight clojure %}
{% raw %}
(defprotocol UnionFind
  (parent
    [this x]
    "Returns the parent of x. Guaranteed to be non-nil.
    If x is a root then the parent of x is x.")
  (rank [this x]
    "Return the rank of x in the current UnionFind.
    The rank measures the height of the tree at the given element.")
  (find-root [this x]
    "Returns the root of the given element.")
  (union [this x y]
    "Merges the subtrees of the trees that x and y belong to."))
{ endraw %}
{% endhighlight %}

For the purposes of this implementation I have added two additional operations: `parent` and `rank`.
This is just to make the code a bit more readable but it's specific to a `RankedUnionFind` which is
one of many ways in which a UnionFind can be implemented.

A `RankedUnionFind` holds a set of trees. Every node in the tree has 2 things: a rank (which is
basically the hight of the tree as measured from that point) and an element (in our case, an evil
gangster). The `parent` operation will return, given an element `x` the element's parent in the tree.
`find-root` then returns the root of the tree and `union` will merge to trees together.

#### Trees!?!?! I thought union find was all about sets

A `RankedUnionFind` represents sets as trees. An element is said to be in the set if it is in the tree.
Two sets are considered the same if their root is the same.

## RankedUnionFind

And now, without further delay, I give you the `RankedUnionFind`

{% highlight ruby %}
{% raw %}
(defrecord RankedUnionFind
  [;; parent-map holds a mapping of element => (parent element).
   ;; every element must have a parent at all times.
   parent-map
   ;; rank-map holds a mapping of element => rank.
   ;; Every element has a rank >= 0
   rank-map]
  UnionFind
  (parent [this x]
    ;; Simply return whatever is mapped by the parent-map
    (get parent-map x))

  (rank [this x]
    ;; Simply return whatever is mapped by the rank-map
    (get rank-map x))

  (find-root [this x]
    ;; start at the given node and recursively
    ;; iterate through all the node's parents until a root
    ;; is found. This operation takes O(log(n)) = O(height(x)) = O(max rank)
    (loop [node x]
      (let [parent* (parent this node)]
        ;; guaranteed to happen eventually since we are traversing a tree
        (if (= parent* node)
          node
          (recur parent*)))))

  (union [this x y]
    (let [root-x (find-root this x)
          root-y (find-root this y)]
      (cond
        (= root-x root-y)
          this
        (> (rank this root-x) (rank this root-y))
          (new RankedUnionFind (assoc parent-map root-y root-x) rank-map)
        (< (rank this root-x) (rank this root-y))
          (new RankedUnionFind (assoc parent-map root-x root-y) rank-map)
        :else
          (RankedUnionFind. (assoc parent-map root-x root-y)
                            (update rank-map root-y inc))))))
{% endraw %}
{% endhighlight %}

Notice that this implementation holds 2 maps:
1. The first one `parent-map` is a hashmap that maps elements to their parent.
   The astute reader will say: "Well, why not use a `Node` which holds a reference to their parent instead?"
   While this sounds very reasonable, you will get into problems when merging to trees
   and you need to change the root of the tree. Take a look at the implementation
   of `union` and see why this can happen.
2. The secon one `rank-map` holds a reference to the rank of each element in the map.
   The reason for this is mainly because it makes the code a bit cleaner IMHO. I encourage
   you to try different ways in which `RankedUnionFind` could be implemented.

Another thing to notice here is that `find-root` determines that a node has is a root node when
the node's parent is itself. I could have also done this be making the node's parent be the `nil`
value, but then the data structure would not support `nil`s.

Finally let's provide a way for users of the data structure to get a new instance

{% highlight ruby %}
{% raw %}
(defn create-ranked-union-find
  "Take a collection of elements and initialize a RankedUnionFind with every
  element mapped to a singleton."
  [coll]
  (let [parents (mapcat (fn [x] [x x]) coll)
        ranks (mapcat (fn [x] [x 0]) coll)]
    (RankedUnionFind. (apply hash-map parents)
                      (apply hash-map ranks))))
{% endraw %}
{% endhighlight %}

# Conclusions

So that's a functional immutable implementation of `UnionFind` in clojure. Notce that if
you strip out comments there arae actually only 30 lines of code (or 40 if you count the protocol)
which just goes to show how concise clojure code can be. Out implementation, although not optimized,
is still fairly efficient. It merges sets in `O(1)` and can `find` an element in a blazingly fast `O(log n)`.
