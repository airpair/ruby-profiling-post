Is your app slow? Stop wasting your time trying random things. You can measure your code to learn where to focus your efforts. Measuring performance is also know as benchmarking or profiling.

## Getting started

The easiest way to profile some Ruby code is saving the current time, execute our code, and then comparing the time it took to complete. There is a small drawback: it is only useful for a specific portion of our code, like a method. This is how we would do it:


```ruby
def expensive_method
  50000.times { |n| n * n }
end

def measure_time
  before = Time.now
  yield
  puts Time.now - before
end

measure_time { expensive_method }

> 0.639588159
```

Another thing we may want to do is compare similar functions, for example is **=~** faster than **.match**? We can find out using the Benchmark/ips (iterations per second) module.

```bash
gem install benchmark-ips
```

```ruby
require 'benchmark/ips'

str = "test"
Benchmark.ips do |x|
  x.report("=~") { str =~ /test/ }
  x.report(".match") { str.match /test/ }
  x.compare!
end
```

When we run this code we will get an output similar to this, which shows clearly that **.match** is the inferior method in terms of performance.

![](http://i.imgur.com/Lh1laJl.png)

> **Info**: Also try out benchmark-bigo which can produce graphs of the methods under test.

## Using ruby-prof

Instead of profiling a few methods against each other we can just profile everything. Ruby's built-in profiler doesn't produce the best results so we are going to install the **ruby-prof** gem.

```bash
gem install ruby-prof
```

Now we can run it like this and it will show us every method that takes more than 5% of the total run-time.

```bash
ruby-prof -m 5 <script.rb>
```

![](http://i.imgur.com/6WEFpEE.png)

A better way to visualize this would be using **qcachegrind**. With **ruby-prof** and the call_tree option we can get cachegrind files that can be used for analysis.

```bash
ruby-prof -p call_tree -f /tmp/test.grind <script.rb>
```

![](http://i.imgur.com/9f7Pae7.png)

## Optimization example

Now it would be a good time for an example of how to optimize some ruby code using profiling. We are going to work with this naïve sorting function:

```ruby
def my_sort(nums, sorted = [])

  min = nums.min

  sorted.push(min)
  nums.delete_if { |n| n == min }

  if nums.size > 0
    my_sort nums, sorted
  else
    sorted
  end
end

my_sort (1..500).to_a.shuffle```


We can load the results on qcachegrind and then we want to sort by the 'self' column which is the total time a specific function took to execute. In this case we can see that **Array#delete_if** stands out.

Now turn your attention to the call graph on the right as it will help us understand the situation, **delete_if** calls **Fixnum#==** 500 times, which means we are going through our whole array to find the number to delete and that's not very efficient.

> **Info**: You can find the official Ruby documentation at http://ruby-doc.org/ it's very useful to have it on your bookmark's bar. If prefer offline documentation try Zeal.

Let's see if we can improve this, a quick look at the Ruby documentation of the Array class and we can find the **delete** method. This method is a little better than **delete_if** since it will stop checking once it finds the number it needs to remove.

It would be even better if we could just remove the sorted item in one step instead of having to search the array at all. Just after the _delete_ method in the documentation we will find the _delete_at_ method, which has the following description:

> "Deletes the element at the specified index, returning that element, or nil if the index is out of range."

Sounds like what we want, but how do we get the index? We could use the _index_ method, but that would defeat the purpose since it has to search the array to find the index.

What we really need is a custom **.min** function that will return the index in addition to the number.

This is the code after:

```ruby
def my_min(nums)
  min = nums.first
  index = 0

   nums.each_with_index do |n, idx|
     if n < min
       min = n
       index = idx
     end
   end

  [min, index]
end

def my_sort(nums, sorted = [])

  min, index = my_min(nums)

  sorted.push(min)
  nums.delete_at(index)

  if nums.size > 0
    my_sort nums, sorted
  else
    sorted
  end
end

my_sort (1..500).to_a.shuffle```

If we profile the modified code we will get the following results:

![](http://i.imgur.com/y2ds9iY.png)

As you can see on the call graph we managed to cut off the whole Array#delete branch, which saves us a lot of work. The sorting function is now much faster (about 40%).

## More profiling

These tools should be enough for most of your profiling needs, but it's always good to have more tools in your toolbox.

Take a look at the following links:

* **stackprof** profiler https://github.com/tmm1/stackprof
* **rbkit** http://rbkit.codemancers.com/

Both of these profilers require Ruby 2.1+ to run. You may also want to take a look at **rblineprof** https://github.com/tmm1/rblineprof which focuses on profiling individual lines of code instead of methods.

Here is an example of rblineprof, using the rblineprof-report gem for nice output.

![](http://i.imgur.com/lP52MUs.png)

Like this post? Take a look at the [author's blog](http://www.blackbytes.info) for more.

