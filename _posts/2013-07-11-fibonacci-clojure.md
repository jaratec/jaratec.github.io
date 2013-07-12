---
layout: post
title: Fibonacci in Clojure
tags: [clojure basics recursion memoization streams]
---


So, you know some Clojure, and you want to practice a little bit, perhaps solving the [4clojure](http://www.4clojure.com/) problems. One such problem is computing the [Fibonacci numbers](http://en.wikipedia.org/wiki/Fibonacci_numbers).

For this session, I'll suggest to open a new file in your editor and to start the Clojure repl to follow along.

The (naive) recursive approach would produce code like this:
{% highlight clojure %}
(defn fibonacci-naive [n]
  (if (<= n 1) n
      (+ (fibonacci-naive (- n 1)) (fibonacci-naive (- n 2)))))
{% endhighlight %}
You have a stop condition for the cases n=0, n=1, and you define f(n) as the sum of f(n-1) and f(n-2).

You can test the function:
{% highlight clojure %}
(map fibonacci-naive (range 10))
{% endhighlight %}

However, this solution does not work very well for a big n, where big depends on your machine. As you increase n, at one point you will observe that the computation starts to get slow. The reason is that you are doubling the number of computations with every recursive step.
Note that, since the algorithm has two recursive calls, you can't perform a tail-call optimization. We'll come back to this.

One idea would be to store the intermediate computation in order to perform a computation only once. This technique is called memoization. We will use a map structure as a cache/lookup table. This particular approach will require mutation, so we'll use an atom for that.
{% highlight clojure %}
(atom {:0 0
       :1 1})
{% endhighlight %}
The cache is initially populated with the first 2 Fibonacci numbers. Next, we'll need a function to get a computed value for a given number from the cache.
{% highlight clojure %}
(fn [k]
  ((keyword (str k)) @cache)))
{% endhighlight %}
We'll also need a function to store a computed value in the cache (note the mutation).
{% highlight clojure %}
(fn [k v]
  (swap! cache assoc (keyword (str k)) v))
{% endhighlight %}
With these in place, we can now write a function that computes the value of a particular number and saves the value in the cache . This function assumes all the previous (technically the previous 2 are enough) numbers have been computed and are stored in the cache.
{% highlight clojure %}
(fn [i]
  (let [result (+ (get-val (- i 1))
                 (get-val (- i 2)))]
   (do
     (put-val i result)
     result)))
{% endhighlight %}
get-val retrieves the value from the cache, put-value saves teh computed value to the cache.
We'll put all these together as bindings in a let expression. We will compute the first n numbers with a for expression.
{% highlight clojure %}
(defn fibonacci-with-state [n]
  (let [cache (atom {:0 0
                     :1 1})
        get-val (fn [k]
                  ((keyword (str k)) @cache))
        put-val (fn [k v]
                  (swap! cache assoc (keyword (str k)) v))
        next (fn [i]
               (let [result (+ (get-val (- i 1))
                               (get-val (- i 2)))]
                 (do
                   (put-val i result)
                   result)))]
    (for [x (range 0 (inc n))]
      (let [v (get-val x)]
        (if (nil? v) (next x)
            v)))))
{% endhighlight %}

You can test safely this version on a big n number. It performs well, because any computation is done only once, and then is stored in memory in case it is needed.
Having arrived at this point, there is one important comment to make. Unlike the naive recursive version where we started the computation at the given n and descended towards 1 and 0, this memoized version (actually the for expression) starts from 0 and goes up toward the given n. Because of this, even a memoized version of fibonacci-naive won't perform, since the the needed compuations (the previous values) are not available in memory. You can try to see for yourself.
{% highlight clojure %}
((memoize fibonacci-naive) 10) ;; try increasing values for n
{% endhighlight %}

The memoized solution is quite lengthy, it also uses mutations. Maybe there is a better way? Of course it is. I only took you on this detour to show you how to implement memoization in case you need it.

We have seen that the memoized version started from 0 and worked its way to the n. We will make a recursive version that will do the same, start from 0 and store the intermediary results. The trick is to use an accumulator as an argument that you will pass along each recursive step.
{% highlight clojure %}
(defn fibonacci-from-the-ground-up [n]
  (letfn [(next-val [c]
            (conj c (+ (last c) (last (butlast c)))))
          (fibonacci-seq [c i]
            (if (> i n) c
                (fibonacci-seq (next-val c) (inc i))))]
    (take n (fibonacci-seq [0 1] 2))))
{% endhighlight %}
We have defined a helper function, next-val. This function takes the so-far computed Fibonacci numbers (a sequence), computes the next Fibonacci number (by using the last two elements of the input), and returns a new sequence where the computed number is conjoined to the input sequence.
Our recursive function, fibonacci-seq takes now 2 arguments. The first argument is the solution (or the partial solution), the second argument is a number to used in the stopping condition. Once we have reached the desired number n, we simply return the accumulated solution.

Before going further, I'd like to take the time to make two comments about this solution. One, the recursive functions that employ the accumulator technique are amenable to tail-call optimizations. For that, the recursive call needs to be the last computation in the recursive step. I won't belabor the point further. And second, this technique is quite appreciated by programmers coming from imperative languages, but for the wrong reasons. To them this accumulation technique resambles mutation, and as a consequence of finding a familiar ground, they tend to overuse it.

Next, let's get rid of the recursion. Being able to replace recursion with higher-order functions is key to writing more expressive programs. Now, if you'll think about it, we already are building the solution from the ground up. So we can use a reduce function that builds/accumulates the solution. The tricks to use, is to use a range for the numbers upto n, and to have as start value the sequence [0 1], same as the recursive solution.
{% highlight clojure %}
(defn fibonacci-reduce [n]
  (let [next-val (fn [c _]
                   (conj c (+ (last c) (last (butlast c)))))]
    (take n (reduce next-val [0 1] (range 2 (inc n))))))
{% endhighlight %}

So far, so good. Let's try writing the Fibonacci numbers as an infinte sequence. Instead of reduce we will use iterate. This function takes as arguments a function f, a starting value x and produces the infinite sequence x, f(x), f(f(x)), f(f(f(x))), ...
So let's transform the reduce version to an iterate version.
{% highlight clojure %}
(defn fibonacci-iterate [n]
  (let [next-val (fn [c]
                   (conj c (+ (last c) (last (butlast c)))))]
    (last (take n (iterate next-val [0 1])))))
{% endhighlight %}

OK. Let me explain now how a lazy sequence works. What is needed is something that produces the next element of the sequence but at the same time it inhibits the evaluation of the rest of the sequence. How should a function that does precisely that, look like? A function should produce the next element and a lazy structure (for example an unevaluated function or a lazy sequence). Here is an example:
{% highlight clojure %}
(def fibonacci-generator
  ((fn generator [a b] (lazy-seq (cons a (generator b (+ a b))))) 0 1))
{% endhighlight %}
Notice that the generator function does not accumulate the fibonacci numbers anymore. It only takes the last two values, needed to compute the next one. Notice also that fibonacci-generator is a function (generator).
This version of fibonacci is deceptively simple.
{% highlight clojure %}
(take 10 fibonacci-generator)
{% endhighlight %}

The end of this post is already overdue, so as a reward for your patience I'll give you a version of fibonacci recursive sequence.
{% highlight clojure %}
(def fibs (concat [0 1] (map + fibs (rest fibs))))
{% endhighlight %}
The function map returns a lazy sequence that is only realized on demand.

{% highlight clojure %}
(take 10 fibs)
{% endhighlight %}

So, we have used the Fibonacci numbers as an excuse to explore some techniques of functional programming in general and of Clojure in particular: recursive functions, memoization and lazy inifinte sequences. Go forth and practice your newly acquired skills.

