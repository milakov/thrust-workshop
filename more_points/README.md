# More Points

Now that we've had our [fun with points](../fun_with_points), it's time to get
serious. Let's use Thrust to get some work done by organizing our points into a
spatial data structure. In this post, we'll become familiar with algorithms
such as `exclusive_scan` and `lower_bound` to build sophisticated data
structures in parallel. Just like before, we'll structure the code such that it
can run anywhere we have parallel resources.

In [exercise.cu](exercise.cu), we have a C++ program that generates some random
two-dimensional points, finds the bounding box of those points, and then
iteratively subdivides that box to build a [hierarchical tree structure](http://en.wikipedia.org/wiki/Quadtree). We
could use this kind of data structure later for performing spatial queries like
[detecting collisions](http://en.wikipedia.org/wiki/Collision_detection), or
[finding nearby neighbors](http://en.wikipedia.org/wiki/K-nearest_neighbor_algorithm),
but for now we'll keep it simple and concentrate on just building the thing.

At a high level, the program looks like this:

    int main()
    {
      const int num_points = 10000000;
      int max_level = XXX;
      int threshold = YYY;

      std::vector<float2> points(num_points);
      generate_random_points(points);

      bbox bounds = compute_bounding_box(points);

      std::vector<int> tags(num_points);
      compute_tags(points, bounds, max_level, tags);

      std::vector<int> indices(num_points);
      sort_points_by_tag(tags, indices);

      std::vector<int> nodes;
      std::vector<int2> leaves;
      build_tree(tags, bounds, max_level, threshold, nodes, leaves);
    }

We'll describe what's going on with the `tags` later.

Our tree data structure is just an array of nodes and a list of leaves. Each leaf indexes a contiguous piece of the `indices` array.

The implementation of `build_tree` function is fairly complex. We'll peek inside later.

# Massaging the Data

Before we can dive into building our tree, we first need to generate our input and massage the data into a format which makes tree construction easy.

## Generating the Points

First, we'll generate some random 2D points in parallel just like we did in the [`fun_with_points`](../fun_with_points) example:

    std::vector<float2> points(num_points);

    generate_random_points(points);

Inside of `generate_random_points` is just a call to `thrust::tabulate` which produces random points.

In the sequential version of the code we keep our `points` data in a `std::vector`. Now that we know about the magic of `thrust::device_vector`, we'll port instances of `std::vector` to `thrust::device_vector` as we go along.

## Bounding the Points

The next thing we do is compute a ["bounding box"](http://en.wikipedia.org/wiki/Minimum_bounding_box) for our points. If you're unfamiliar with the idea, a bounding box is a box which contains or "bounds" all of our points. You can think of it as describing the geometric boundaries of our problem set.

For our purposes, a `bbox` is just two points which specify the coordinates of the extremal corners of the box along the two coordinate axes x and y:

    struct bbox
    {
      float xmin, xmax;
      float ymin, ymax;
    
      // initialize empty box
      inline __host__ __device__
      bbox() : xmin(FLT_MAX), xmax(-FLT_MAX), ymin(FLT_MAX), ymax(-FLT_MAX)
      {}
      
      // initialize a box containing a single point
      inline __host__ __device__
      bbox(const float2 &p) : xmin(p.x), xmax(p.x), ymin(p.y), ymax(p.y)
      {}
    };

It's defined in [`util.h`](util.h) in the source.

In order to compute a single box which is large enough to contain all of our points, we need to inspect them all and __reduce__ them into a single value -- the box.

Here's what the sequential code looks like:

    // start with an empty box
    bbox bounds;

    // incrementally enlarge the box to include each point
    for(int i = 0; i < num_points; ++i)
    {
      float2 p = points[i];
      if(p.x < bounds.xmin) bounds.xmin = p.x;
      if(p.x > bounds.xmax) bounds.xmax = p.x;
      if(p.y < bounds.ymin) bounds.ymin = p.y;
      if(p.y > bounds.ymax) bounds.ymax = p.y;
    }

We just loop through the points and make sure the box is large enough to contain each one. If the box isn't large enough along a particular dimension, we extend it such that it is just large enough to contain the point.

At first glance, it may seem difficult to parallelize this operation because each iteration incrementally builds off of the last one. In fact, it's possible to cast this operation as a __reduction__.

In the [`fun_with_points`](../fun_with_points) example, we used `thrust::reduce` to compute the average of a collection of points. Here, the result was the same type as the input -- the average of a collection of points is still a point.

In this case, we'd like to compute a result (a `bbox`) which is a different type than the input (a collection of `float2`s). That's okay -- as long as the type of the input is convertible to the result (note the second constructor of `bbox`) `thrust::reduce` will make it work.

To implement this bounding box reduction, we'll introduce a functor which merges two `bbox`s together as the fundamental reduction step. The resulting `bbox` is large enough to hold the two inputs:

    struct merge_boxes
    {
      inline __host__ __device__
      bbox operator()(const bbox &a, const bbox &b) const
      {
        bbox result;
        result.xmin = min(a.xmin, b.xmin);
        result.xmax = max(a.xmax, b.xmax);
        result.ymin = min(a.ymin, b.ymin);
        result.ymax = max(a.ymax, b.ymax);
        return result;
      }
    };

Internally, the way the reduction will work is to create, for each point, a single `bbox` which will bound only that point. Then, the reduction will merge `bbox`s in pairs until finally a single one which bounds everything results:

    bbox compute_bounding_box(const thrust::device_vector<float2> &points)
    {
      // we pass an empty bounding box for thrust::reduce's init parameter
      bbox empty;
      return thrust::reduce(points.begin(), points.end(), empty, merge_bboxes());
    }

## Linearizing the Points

The next step in preparing our data for tree construction is to augment its representation. The basic idea behind the tree construction process is to pose it as a spatial sorting problem. But sorts are one dimensional and we have two dimensional data. What does it mean to sort 2D points?

Since we want to organize our points spatially, we'd like a sorting solution which preserves spatial locality. In other words, we want points that are nearby in 2D to be near each other in the one dimensional `points` array.

This is actually a whole lot easier than it sounds. The basic idea is to "tag" each point with its [index](http://en.wikipedia.org/wiki/Morton_code) along a [space-filling curve](http://en.wikipedia.org/wiki/Space_filling_curve).

It turns out that points with nearby tags are also nearby in 2D! That means if we sort our collection of `points` by their `tags`, we'll order them in a way that encourages spatial locality which will be important to the tree building process later.

The sequential code is pretty simple. It just associates with each point a tag:

    std::vector<int> tags(points.size());

and computes them with the `compute_tags` function:

    void compute_tags(const std::vector<float2> &points,
                      const bbox &bounds,
                      int max_level,
                      std::vector<int> &tags)
    {
      for(int i = 0; i < points.size(); ++i)
      {
        float2 p = points[i];
        tags[i] - point_to_tag(p, bounds, max_level);
      }
    }

The `point_to_tag` computation takes a point `p`, the `bounds` of the entire collection of `points`, and the index of the tree's `max_level` and computes the point's spatial code. If you're interested in the details, you can peek inside [`util.h`](util.h) where it's defined.

This operation looks a lot like the point classification problem from the [`fun_with_points`](../fun_with_points) exercise. We know that we can parallelize embarrassingly parallel operations like these with `thrust::transform`:

    struct classify_point
    {
      bbox box;
      int max_level;

      classify_point(const bbox &bounds, int max_level) :
        box(bounds),
        max_level(max_level)
      {}

      inline __device__ __host__
      int operator()(const float2 &p)
      {
        return point_to_tag(p, box, max_level);
      }
    };

    void compute_tags(const thrust::device_vector<float2> &points,
                      const bbox &bounds,
                      std::vector<int> &tags)
    {
      thrust::transform(points.begin(), points.end(),
                        tags.begin(),
                        classify_point(bounds, max_level));
    }

The only thing we need to do is introduce the `classify_point` functor whose job it is to call the `point_to_tag` function.

## Sorting by Tag

Now that each point has a spatial `tag`, we can organize them spatially just by sorting the `points` by their `tags`.

Since sorting the `points` directly would destroy their original order, we'll introduce one level of indirection and sort their `indices` within the list instead.

The sequential CPU code does it this way:

    struct compare_tags
    {
      template <typename Pair>
      inline bool operator()(const Pair &p0, const Pair &p1) const
      {
        return p0.first < p1.first;
      }
    };

    void sort_points_by_tag(std::vector<int> &tags, std::vector<int> &indices)
    {
      // introduce a temporary array of pairs for sorting purposes
      std::vector<std::pair<int,int> > tag_index_pairs(num_points);
      for(int i = 0; i < num_points; ++i)
      {
        tag_index_pairs[i].first = tags[i];
        tag_index_pairs[i].second = i;
      }

      std::sort(tag_index_pairs.begin(), tag_index_pairs.end(), compare_tags());

      // copy sorted data back into input arrays
      for(int i = 0; i < num_points; ++i)
      {
        tags[i]    = tag_index_pairs[i].first;
        indices[i] = tag_index_pairs[i].second;
      }
    }

Which is a pretty roundabout way of coaxing a key-value sort out of `std::sort`. With Thrust we can do it in parallel with just a call to `thrust::sort_by_key`:

    void sort_points_by_tag(thrust::device_vector<int> &tags, thrust::device_vector<int> &indices)
    {
      thrust::sequence(indices.begin(), indices.end());
      thrust::sort_by_key(tags.begin(), tags.end(), indices.begin());
    }

# Building the Tree

Now that we've got our points nice and organized, it's time to build the tree! We'll build each level of the tree one by one, and building each level requires a series of steps.

Here's the high-level overview of the process:

    void build_tree(const std::vector<int> &tags,
                    const bbox &bounds,
                    int max_level,
                    int threshold,
                    std::vector<int> &nodes,
                    std::vector<int2> &leaves)
    {
      std::vector<int> active_nodes(1,0);
      
      // build the tree one level at a time, starting at the root
      for(int level = 1; !active_nodes.empty() && level <= max_level; ++level)
      {
        // each node has four children since this is a quad tree
        std::vector<int> children(4 * active_nodes.size());

        compute_child_tag_masks(active_nodes, level, max_level, children);

        std::vector<int> lower_bounds(children.size());
        std::vector<int> upper_bounds(children.size());
        find_child_bounds(tags, children, level, max_level, lower_bounds, upper_bounds);

        // mark each child as either empty, an interior node, or a leaf
        std::vector<int> child_node_kind(children.size(), 0);
        classify_children(children, lower_bounds, upper_bounds, level, max_level, threshold, child_node_kind);

        // enumerate the nodes and leaves at this level
        std::vector<int> nodes_on_this_level(child_node_kind.size());
        std::vector<int> leaves_on_this_level(child_node_kind.size());

        std::pair<int,int> num_nodes_and_leaves_on_this_level =
          enumerate_nodes_and_leaves(child_node_kind, nodes_on_this_level, leaves_on_this_level);

        create_child_nodes(child_node_kind, nodes_on_this_level, leaves_on_this_level, leaves.size(), nodes);

        create_leaves(child_node_kind, leaves_on_this_level, lower_bounds, upper_bounds, num_nodes_and_leaves_on_this_level.second, leaves);

        activate_nodes_for_next_level(children, child_node_kind, active_nodes);

        activate_nodes_for_next_level(children, child_node_kind, num_nodes_and_leaves_on_this_level.first, active_nodes);
      }
    }

You can see that it takes as input the information we computed in the prior
steps (`tags`, `bounds`) and some tweakable knobs (`max_level`, `threshold`)
and produces two arrays: `nodes` and `leaves`.

Each element of the `nodes` array is an index which identifies whether the node is empty, or whether it
refers to a terminal leaf, or an interior node. When it is a leaf, the index encodes where in the `leaves` array it lives.
When it is an interior node, it stores the index of its first child so that we can find it later when we traverse the tree.

As we build the tree level by level, we'll keep a list of "active" nodes. These
are the tags of the nodes on the current level. In each iteration, our job is
to search for and construct nodes for their children.

We start out at the root. The root of the tree has a tag of `0`:

    std::vector<int> active_nodes(1,0);

## Masking off the Search

In order to search through the `tags` array for the `active_nodes`'s children,
we need to "mask off" the search area of interest. We do that by taking
each tag in the `active_nodes` array, and producing a mask. Geometrically,
this tag mask corresponds to a quarter of the box spanned by the active
node. Since there are four quarters of the box, we create room for four `children` for each of the `active_nodes`, and call `compute_child_tag_masks`:

    std::vector<int> children(4 * active_nodes.size());
    compute_child_tag_masks(active_nodes, level, max_level, children);

Inside, `compute_child_tag_masks` looks like this:

    void compute_child_tag_masks(const std::vector<int> &active_nodes,
                                 int level,
                                 int max_level,
                                 std::vector<int> &children)
    {
      for (int i = 0 ; i < active_nodes.size() ; ++i)
      {
        int tag = active_nodes[i];
        children[4*i+0] = child_tag_mask(tag, 0, level, max_level);
        children[4*i+1] = child_tag_mask(tag, 1, level, max_level);
        children[4*i+2] = child_tag_mask(tag, 2, level, max_level);
        children[4*i+3] = child_tag_mask(tag, 3, level, max_level);
      }
    }

For each of the `active_nodes`, we get its `tag` and call the helper function
`child_tag_mask` to compute a mask for each one of its four `children`. To
parallelize this operation with Thrust, we'll need to find a way to "expand" a collection
of items, as the output (`children`) is four times the size of the input
(`active_nodes`). In other words, it won't be a simple call to `thrust::transform`.

Instead of parallelizing over the elements of the input array, an alternative
approach could parallelize over the output. For each output element in
`children`, we could look up the corresponding input element in `active_nodes`.
If we know the index `idx` of each output element from `children`, we can get
the index of the corresponding input element easily -- it's just `idx/4`.
Luckily, we know how to generate indices for each output element --
`thrust::tabulate` does this for us. Since the interface of `thrust::tabulate`
only takes a single output range, we'll have to go "out of band" to get the tag
of each child's parent node.

Let's look at the whole function, which we've ported to use `thrust::device_vector`:

    struct child_index_to_tag_mask
    {
      int level, max_level;
      thrust::device_ptr<const int> nodes;
      
      child_index_to_tag_mask(int lvl, int max_lvl, thrust::device_ptr<const int> nodes) : level(lvl), max_level(max_lvl), nodes(nodes) {}
      
      inline __device__ __host__
      int operator()(int idx) const
      {
        int tag = nodes[idx/4];
        int which_child = (idx&3);
        return child_tag_mask(tag, which_child, level, max_level);
      }
    };
    
    void compute_child_tag_masks(const thrust::device_vector<int> &active_nodes,
                                 int level,
                                 int max_level,
                                 thrust::device_vector<int> &children)
    {
      thrust::tabulate(children.begin(), children.end(),
                       child_index_to_tag_mask(level, max_level, active_nodes.data()));
    }

We call `thrust::tabulate` with `children` as our output array and use
`child_index_to_tag_mask` as our functor.  In addition to `level`, and
`max_level`, which are needed to call the helper function `child_tag_mask`
inside the functor, we also use the special member function
`active_nodes.data()` to pass a pointer to the `active_nodes` array into our
functor. This is how we go out of band into the `active_nodes` array -- even
though `thrust::tabulate` doesn't know about `active_nodes`, we can still use
it as input!

`active_nodes.data()` returns a special type of pointer called a
`thrust::device_ptr` which keeps track of the fact that the data it points to
lives on the GPU. This makes accessing data on the GPU easy.

Inside of `child_index_to_tag_mask`, after receiving the `idx` of the child, we
first look into the `nodes` array to find the `tag` of the parent node. Next,
we use `idx` again to figure out which child of the node (0,1,2, or 3)
we're computing. Finally, we're able to call `child_index_to_tag_mask` just
like the sequential version of the code.

## Searching for Children

Now that we know where to look, we can search for the list of tags spanned or bound by each active node's children. We'll
represent the lists as a couple of arrays:

    std::vector<int> lower_bounds(children.size());
    std::vector<int> upper_bounds(children.size());

For each element of `children`, `lower_bounds` stores the index of the first
tag it spans inside of `tags`. Likewise, `upper_bounds` stores the index of the
tag one past the last spanned. In other words, for each element of `children`,
we'll have the boundaries of a contiguous span of elements inside `tags`.

Inside of `find_child_bounds`, we do the search:

    void find_child_bounds(const std::vector<int> &tags,
                           const std::vector<int> &children,
                           int level,
                           int max_level,
                           std::vector<int> &lower_bounds,
                           std::vector<int> &upper_bounds)
    {
      int length = (1 << (max_level - level) * 2) - 1;
      for (int i = 0 ; i < children.size() ; ++i)
      {
        lower_bounds[i] = std::lower_bound(tags.begin(), tags.end(), children[i]) - tags.begin();
        
        upper_bounds[i] = std::upper_bound(tags.begin(), tags.end(), children[i] + length) - tags.begin();
      }
    }

For each mask element of `children`, we do a binary search in the `tags` array to discover the indices of the tags it spans.
In the C++ standard library, these searches are implemented with `std::lower_bound` and `std::upper_bound`.

Parallelizing this operation is pretty simple, as Thrust provides "vectorized" versions of `lower_bound` and `upper_bound`. We split the `for` loop into two separate operations:

    void find_child_bounds(const thrust::device_vector<int> &tags,
                           const thrust::device_vector<int> &children,
                           int level,
                           int max_level,
                           thrust::device_vector<int> &lower_bounds,
                           thrust::device_vector<int> &upper_bounds)
    {
      thrust::lower_bound(tags.begin(),
                          tags.end(),
                          children.begin(),
                          children.end(),
                          lower_bounds.begin());
      
      int length = (1 << (max_level - level) * 2) - 1;
    
      using namespace thrust::placeholders;
    
      thrust::upper_bound(tags.begin(),
                          tags.end(),
                          thrust::make_transform_iterator(children.begin(), _1 + length),
                          thrust::make_transform_iterator(children.end(), _1 + length),
                          upper_bounds.begin());
    }
                           
First, we call `thrust::lower_bound` to search the collection of `tags` for each element in the `children` collection.

Next, to compute `upper_bounds`, we call `thrust::upper_bound` similarly. This
time, to incorporate `length` into each element of `children`, we create a
`transform_iterator` using a placeholder expression.

## Classification

Now that we know for each child the sublist of of `tags` it spans, we can
classify whether each child is empty, an interior node, or a terminal leaf of
our tree. For each element of the `children` array, we'll store an `int` to keep track of its kind:

    std::vector<int> child_node_kind(children.size(), 0);
    classify_children(lower_bounds, upper_bounds, level, max_level, threshold, child_node_kind);

Classifying each child is simple, as it only depends on the number of tag elements each
child spans. The sequential code is just another `for` loop:

    void classify_children(const std::vector<int> &lower_bounds,
                           const std::vector<int> &upper_bounds,
                           int level,
                           int max_level,
                           int threshold,
                           std::vector<int> &child_node_kind)
    {
      for(int i = 0; i < upper_bounds.size(); ++i)
      {
        int count = upper_bounds[i] - lower_bounds[i];
        if(count == 0)
        {
          child_node_kind[i] = EMPTY;
        }
        else if(level == max_level || count < threshold)
        {
          child_node_kind[i] = LEAF;
        }
        else
        {
          child_node_kind[i] = NODE;
        }
      }
    }

The number of tags (`count`) spanned by each child `i` is just the difference between that child's upper and lower bounds.
An `EMPTY` child corresponds to a `count` of zero. Then, depending on the `threshold`, a child is either a `NODE` or a `LEAF`.

We've parallelized so many of this kind of loop that it should be second nature by now. It's just `thrust::transform`:

    struct classify_node
    {
      int threshold;
      int last_level;
      
      classify_node(int threshold, int last_level) : threshold(threshold), last_level(last_level) {}
    
      inline __device__ __host__
      int operator()(int lower_bound, int upper_bound) const
      {
        int count = upper_bound - lower_bound;
        if (count == 0)
        {
          return EMPTY;
        }
        else if (last_level || count < threshold)
        {
          return LEAF;
        }
        else
        {
          return NODE;
        }
      }
    };
    
    void classify_children(const thrust::device_vector<int> &lower_bounds,
                           const thrust::device_vector<int> &upper_bounds,
                           int level,
                           int max_level,
                           int threshold,
                           thrust::device_vector<int> &child_node_kind)
    {
      thrust::transform(lower_bounds.begin(), lower_bounds.end(),
                        upper_bounds.begin(),
                        child_node_kind.begin(),
                        classify_node(threshold, level == max_level));
    }

## Ranking Nodes and Leaves

Now that we've classified each of this level's children as either nodes or
leaves, we need to know for each node which node that it is. In other words, we
need to label the first node with a `0`, the second node with a `1`, the third
with a `3`, and so on. We also need to do the same thing for leaves. "Ranking"
each child in this way will tell us where we need to put it in our final data
structure.

We'll introduce two new arrays to store these ranks and also tally total the number of nodes and leaves on this level:

    std::vector<int> nodes_on_this_level(child_node_kind.size());
    std::vector<int> leaves_on_this_leveL(child_node_kind.size());

    std::pair<int,int> num_nodes_and_leaves_on_this_level =
      enumerate_nodes_and_leaves(child_node_kind, nodes_on_this_level, leaves_on_this_level);

Let's look at the sequential code:

    std::pair<int,int> enumerate_nodes_and_leaves(const std::vector<int> &child_node_kind,
                                                  std::vector<int> &nodes_on_this_level,
                                                  std::vector<int> &leaves_on_this_level)
    {
      for(int i = 0, prefix_sum = 0; i < child_node_kind.size(); ++i)
      {
        nodes_on_this_level[i] = prefix_sum;
        if(child_node_kind[i] == NODE)
        {
          ++prefix_sum;
        }
      }
    
      for(int i = 0, prefix_sum = 0; i < child_node_kind.size(); ++i)
      {
        leaves_on_this_level[i] = prefix_sum;
        if(child_node_kind[i] == LEAF)
        {
          ++prefix_sum;
        }
      }
    
      std::pair<int,int> num_nodes_and_leaves_on_this_level;
    
      num_nodes_and_leaves_on_this_level.first = nodes_on_this_level.back() + (child_node_kind.back() == NODE ? 1 : 0);
      num_nodes_and_leaves_on_this_level.second = leaves_on_this_level.back() + (child_node_kind.back() == LEAF ? 1 : 0);
    
      return num_nodes_and_leaves_on_this_level;
    }

The sequential version of the code loops through the `child_node_kind` array
twice. Each time it encounters a `NODE` it increments a counter called
`prefix_sum`. In each iteration, the counter gets stored to the corresponding
element of `nodes_on_this_level`. The same thing happens for the leaves. The
result is that the `nodes_on_this_level` array contains an ascending sequence
of integers, starting at zero. The locations where the value of the sequence
increments are at locations in `child_node_kind` which correspond to a node.

This means that the last element of the `nodes_on_this_level` array is one less
than the total number of interior nodes. To find the total, we take this number
and add one if the last element of `child_node_kind` corresponds to a `NODE`
(and do the same computation for the leaves).

This kind of loop seems really hard to parallelize because the value of our
counter depends on elements of `child_node_kind` we encountered in the past.
On the other hand, the reduction loop from the
[`fun_with_points`](../fun_with_points) example seemed the same way at first.
Maybe there's a way to compute several sums in parallel?

It turns out that the operation that this `for` loop implements is called a
"prefix sum" (hence the name of our counter) or a "scan". A scan is kind of
like a reduction, but instead of producing just a single result, we associate a
result with each input element which is the sum of elements encountered so far.
Thrust calls this particular flavor an "exclusive scan" because each input is
excluded from its corresponding sum. That is, the counter is updated *after*
the corresponding sum is written to the output.

Let's look at how to use `thrust::transform_exclusive_scan` to parallelize these prefix sums:

    std::pair<int,int> enumerate_nodes_and_leaves(const thrust::device_vector<int> &child_node_kind,
                                                  thrust::device_vector<int> &nodes_on_this_level,
                                                  thrust::device_vector<int> &leaves_on_this_level)
    {
      thrust::transform_exclusive_scan(child_node_kind.begin(), 
                                       child_node_kind.end(), 
                                       nodes_on_this_level.begin(), 
                                       is_a<NODE>(), 
                                       0, 
                                       thrust::plus<int>());
      
      thrust::transform_exclusive_scan(child_node_kind.begin(), 
                                       child_node_kind.end(), 
                                       leaves_on_this_level.begin(), 
                                       is_a<LEAF>(), 
                                       0, 
                                       thrust::plus<int>());
    
      std::pair<int,int> num_nodes_and_leaves_on_this_level;
    
      num_nodes_and_leaves_on_this_level.first = nodes_on_this_level.back() + (child_node_kind.back() == NODE ? 1 : 0);
      num_nodes_and_leaves_on_this_level.second = leaves_on_this_level.back() + (child_node_kind.back() == LEAF ? 1 : 0);
    
      return num_nodes_and_leaves_on_this_level;
    }

You can see that the two loops have collasped into high level algorithm calls, but the code which computes the total sums at the end is unchanged.
Let's decipher what's going on inside one of these calls to `thrust::transform_exclusive_scan`:

    thrust::transform_exclusive_scan(child_node_kind.begin(),
                                     child_node_kind.end(),
                                     nodes_on_this_level.begin(),
                                     is_a<NODE>(),
                                     0,
                                     thrust::plus<int>());

First, we pass the input range, this is just `child_node_kind`. We'll store the
scan to `nodes_on_this_level`, which comes next. The next argument,
`is_a<NODE>()`, is a functor. Whenever the scan encounters an element from
`child_node_kind` which is a `NODE`, this functor will transform the input
element into `true`.  Otherwise, it will return `false`. Next, we tell the
scan that we want to start counting from `0`. Finally, we tell the scan
how to sum two results from `is_a<NODE>()` together: just do an integer `plus` operation. The
second call to `transform_exclusive_scan` for leaves is interpreted
similarly.

## Creating the Child Nodes

Now that we've done all the bookkeeping required, we can create new nodes to
encode the children of the `active_nodes` of this level.  To do this, we'll
append to the end of our `nodes` array a new entry for each child. The value of
each entry will encode the kind of node it is, along with information about
where to find the data associated with it.

The sequential code looks like this:

    void create_child_nodes(const std::vector<int> &child_node_kind,
                            const std::vector<int> &nodes_on_this_level,
                            const std::vector<int> &leaves_on_this_level,
                            int num_leaves,
                            std::vector<int> &nodes)
    {
      int num_children = child_node_kind.size();
    
      int children_begin = nodes.size();
      nodes.resize(nodes.size() + num_children);
      
      for(int i = 0 ; i < num_children; ++i)
      {
        switch(child_node_kind[i])
        {
        case EMPTY:
          nodes[children_begin + i] = get_empty_id();
          break;
        case LEAF:
          nodes[children_begin + i] = get_leaf_id(num_leaves + leaves_on_this_level[i]);
          break;
        case NODE:
          nodes[children_begin + i] = nodes.size() + 4 * nodes_on_this_level[i];
          break;
        }
      }
    }

We begin by reserving space for each new child entry at the end of the `nodes`
array. Before resizing the array, we note the index which marks the beginning
of the new list of child nodes. To create each new entry, we loop through the
new nodes, and depending on the kind of node it is, we encode some bookkeeping
information: either an empty node id, a leaf id, or the index of an interior
node's first child.

Of course, this is another job for `thrust::transform`:

    struct write_nodes
    {
      int num_nodes, num_leaves;
    
      write_nodes(int num_nodes, int num_leaves) : 
        num_nodes(num_nodes), num_leaves(num_leaves) 
      {}
    
      template <typename tuple_type>
      inline __device__ __host__
      int operator()(const tuple_type &t) const
      {
        int node_type = thrust::get<0>(t);
        int node_idx  = thrust::get<1>(t);
        int leaf_idx  = thrust::get<2>(t);
    
        if (node_type == EMPTY)
        {
          return get_empty_id();
        }
        else if (node_type == LEAF)
        {
          return get_leaf_id(num_leaves + leaf_idx);
        }
        else
        {
          return num_nodes + 4 * node_idx;
        }
      }
    };

    void create_child_nodes(const thrust::device_vector<int> &child_node_kind,
                            const thrust::device_vector<int> &nodes_on_this_level,
                            const thrust::device_vector<int> &leaves_on_this_level,
                            int num_leaves,
                            thrust::device_vector<int> &nodes)
    {
      int num_children = child_node_kind.size();
    
      int children_begin = nodes.size();
      nodes.resize(nodes.size() + num_children);
      
      thrust::transform(thrust::make_zip_iterator(
                            thrust::make_tuple(
                                child_node_kind.begin(), nodes_on_this_level.begin(), leaves_on_this_level.begin())),
                        thrust::make_zip_iterator(
                            thrust::make_tuple(
                                child_node_kind.end(), nodes_on_this_level.end(), leaves_on_this_level.end())),
                        nodes.begin() + children_begin,
                        write_nodes(nodes.size(), num_leaves));
    }

The interesting thing about this transformation is that it requires three
inputs. Since `thrust::transform` only supports transformations with up to two
inputs, we "fool" it by zipping together three inputs with a `zip_iterator`.

To create a `zip_iterator`, we call `thrust::make_zip_iterator` with an
argument which is a `tuple` of the iterators we want to zip together. To make
the `tuple` of iterators, we use `thrust::make_tuple`.

The functor we pass to `thrust::transform`, `write_nodes`, unpacks the `tuple`
it receives using the special function `thrust::get<i>`. We call it once with a
different index for each of the three parameters: `node_type`, `node_idx`, and
`leaf_idx`. The rest of the functor body looks like the original body of the
`for` loop.

## Creating the Leaves

We're almost done with creating this level's nodes. However, some of these
nodes are terminal leaves. For these, we'll need to encode which of our
original points they contain. Remember, for each child, we stored indices into
the list of points they bounded inside the `lower_bounds` and `upper_bounds`
arrays. For each node which is a leaf, we'll append these bounds into an
auxiliary array, `leaves`.

Here's the sequential code:

    void create_leaves(const std::vector<int> &child_node_kind,
                       const std::vector<int> &leaves_on_this_level,
                       const std::vector<int> &lower_bounds,
                       const std::vector<int> &upper_bounds,
                       int num_leaves_on_this_level,
                       std::vector<int2> &leaves)
    {
      int children_begin = leaves.size();
    
      leaves.resize(leaves.size() + num_leaves_on_this_level);
      
      for(int i = 0; i < child_node_kind.size() ; ++i)
      {
        if(child_node_kind[i] == LEAF)
        {
          leaves[children_begin + leaves_on_this_level[i]] = make_int2(lower_bounds[i], upper_bounds[i]);
        }
      }
    }

We begin by noting where in the `leaves` array the new children begin. Next, we
reserve enough room in the `leaves` array to append all the new children.
Finally, for each child node which is a leaf, we copy to an entry in `leaves`
an `int2` which encodes the range of points the leaf spans. This is just the
pair of bounds.

The position of each new leaf is encoded by the `leaves_on_this_level` array we
computed previously using a prefix sum. This kind of write is indirect: to find
the location within `leaves` to write to, we do a lookup into
`leaves_on_this_level`. We often call this kind of indirect write a __scatter__
operation.

We can parallelize this operation using `thrust::scatter_if`:

    struct make_leaf
    {
      typedef int2 result_type;
      template <typename tuple_type>
      inline __device__ __host__
      int2 operator()(const tuple_type &t) const
      {
        int x = thrust::get<0>(t);
        int y = thrust::get<1>(t);
    
        return make_int2(x, y);
      }
    };

    void create_leaves(const thrust::device_vector<int> &child_node_kind,
                       const thrust::device_vector<int> &leaves_on_this_level,
                       const thrust::device_vector<int> &lower_bounds,
                       const thrust::device_vector<int> &upper_bounds,
                       int num_leaves_on_this_level,
                       thrust::device_vector<int2> &leaves)
    {
      int children_begin = leaves.size();
    
      leaves.resize(leaves.size() + num_leaves_on_this_level);
    
      thrust::scatter_if(thrust::make_transform_iterator(
                             thrust::make_zip_iterator(
                                 thrust::make_tuple(lower_bounds.begin(), upper_bounds.begin())),
                             make_leaf()),
                         thrust::make_transform_iterator(
                             thrust::make_zip_iterator(
                                 thrust::make_tuple(lower_bounds.end(), upper_bounds.end())),
                             make_leaf()),
                         leaves_on_this_level.begin(),
                         child_node_kind.begin(),
                         leaves.begin() + children_begin,
                         is_a<LEAF>());
    }

This is the gnarliest looking call we've seen so far. The basic idea is that
we're going to take the elements of `lower_bounds` and `upper_bounds` and
conditionally scatter them to the `leaves` array when they correspond to a node
which is a leaf. 

Let's begin by deciphering the `make_transform_iterator` and
`make_zip_iterator` calls. To fit into the `leaves` array which is of type
`int2`, We need to turn the elements of `lower_bounds` and `upper_bounds` into
`int2`. To do that, we begin by zipping together the two arrays, which
basically results in an array of `tuple`s. To get an `int`, we need to
transform the `tuple` using the `make_leaf` functor. If we hook that into
`make_transform_iterator`, we'll have what looks like an array of `int2`, which
is what will fit the `leaves` array.

To find the location in the `leaves` array where each `int2` should go, we pass
our `leaves_on_this_level` array. Remember, this array stores the "rank" of
each leaf node. When these ranks are used as scatter indices, this ensures that
the leaves are stored contiguously at the end of the `leaves` array.

But we don't want to scatter all the bounds to the `leaves` array -- we only
want to store the ones that correspond to `leaves`. That's where our condition
comes in. To compute whether or not an element from our input should be
scattered, we pass the `child_node_kind` array along with the `is_a<LEAF>()`
functor. For each input element, `thrust::scatter_if` will transform the
corresponding element of the `child_node_kind` array using this functor. If it
evaluates to true, then it will scatter the input element. Otherwise,
`thrust::scatter_if` will just ignore it.

Finally, we pass the position in the `leaves` array where the new children
begin: this is at `leaves.begin + children_begin`.

Phew!

## Activating the Next Level

To finish up the iteration, we need to activate the children which will become the parents of the next level.
These nodes are simply the interior nodes we encountered on this level. 

The sequential code simply copies to the `active_nodes` array those elements of `children` which are interior nodes:

    void activate_nodes_for_next_level(const std::vector<int> &children,
                                       const std::vector<int> &child_node_kind,
                                       int num_nodes_on_this_level,
                                       std::vector<int> &active_nodes)
    {
      active_nodes.resize(num_nodes_on_this_level);
      
      for(int i = 0, j = 0; i < children.size(); ++i)
      {
        if(child_node_kind[i] == NODE)
        {
          active_nodes[j++] = children[i];
        }
      }
    }

Hmm... the way we're using that `j` counter looks suspiciously similar to a
prefix sum. However, in this case, we're not writing the value of the counter
to an output array, we're using it to *index* into an output array. This kind
of operation is sometimes called __stream compaction__ which is basically a
prefix sum and scatter combined.

With Thrust, we can implement a stream compaction operation using `thrust::copy_if`:

    void activate_nodes_for_next_level(const thrust::device_vector<int> &children,
                                       const thrust::device_vector<int> &child_node_kind,
                                       int num_nodes_on_this_level,
                                       thrust::device_vector<int> &active_nodes)
    {
      active_nodes.resize(num_nodes_on_this_level);
      
      thrust::copy_if(children.begin(),
                      children.end(),
                      child_node_kind.begin(),
                      active_nodes.begin(),
                      is_a<NODE>());
    }

Here, the idea is that we're going to copy an element from the input `children`
array to the output `active_nodes` array only when the element satisfies a
condition (kind of like `thrust::scatter_if`). Here, the condition is given by
the `is_a<NODE>()` functor, which is applied to each element of the
`child_node_kind` array. Each time an element of the `child_node_kind` array
satisfies the functor, the corresponding element from `children` will get
copied to `active_nodes`.

And that's it! If we iterate these steps up to `max_level` while `active_nodes`
still has work left in it, we'll arrive at a finished tree.

# Performance

Let's see how we did. [`performance.cu`](performance.cu) provides an
instrumented version of the solution we can use for measuring its performance.
Since we've built our solution using `thrust::device_vector`, it's easy to
switch between building a program which targets the CPU or the GPU on the
command line:

    # build the cpu solution
    $ scons cpu_performance
    $ ./cpu_performance
    Warming up...
    
    Timing...
    
    5.27696 millions of points generated and treeified per second.

    # build the gpu solution
    $ scons gpu_performance
    $ ./gpu_performance
    Warming up...
    
    Timing...
    
    144.16 millions of points generated and treeified per second.
    
For the GPU (an NVIDIA Tesla K20c) version, that's over 25 times the
performance as the sequential version of the code running on the CPU (an Intel
    Core i7 860).

Substantial optimizations could probably still be made to both versions of the
code. For example, some temporary `device_vector`s we introduced could probably
be replaced with `transform_iterator`s which don't incur memory storage
overhead. Additionally, some opportunities for fusion exist. For example, both
of our calls to `thrust::transform_exclusive_scan` could be fused together into
a single scan with some clever use of `zip_iterator`s. But those are exercises
left to the reader.

# Wrapping Up

In working through this example, we saw how fundamental parallel algorithms
could be used to accelerate the construction and manipulation of a complex data
structure. However, parallelization can come at a cost in code complexity. The
linear representation of this example is a far cry from the classical
organization of a tree data structure via a recursive arrangement of nodes.
Often, parallelization involves rethinking the organization of our data to make
operating on it in parallel more convenient.

