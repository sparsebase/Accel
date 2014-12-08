Accel
===================
[Jeremy Newlin](http://www.jeremynewlin.info/) and [Danny Rerucha](http://www.dannyrerucha.com/)

This repository contains the functionality for two types of acceleration structures.  A uniform grid for accelerating nearest neighbor searches, and a KD Tree for accelerating ray tracking.  Both are run on the GPU using CUDA.  This project is a joint effort between Danny Rerucha and I for the GPU Programming Course at the University of Pennsylvania (CIS565).

Nearest Neighbor Search and the Uniform Grid
----------
Before we get to our discussion of the uniform grid, let's talk about nearest neighbor search for a second.  It's hopefully somewhat clear what we're trying to accomplish - we want to find some neighboring particles (or photons, points in a cloud, etc) that fall within in a certain radius.  This can be done pretty easily if performance is not a problem.  You can simply loop over all of the particles and compare their distance with the radius.  If the distance is less than the radius, you have a neighbor!  For comparison purposes, we've included implementations of this algorithm in this project.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/neighbor_example.png)
An example of a particle (green) and its neighbors (blue).

But, we ARE interested in performance, and the naive solution isn't going to cut it.  Enter the uniform grid, which is a type of spatial hashing data structure that allows for rapid nearest neighbor searches on the GPU (and CPU).

The basic idea of the structure can be described easily with a picture.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/hash_grid_image.png)

In this example, we can particles [0,1,2,3,4,5] in the grid.  We hash the particles to their grid cell ids into an unsorted list.  Then, we sort that list by grid cell id.  Then, in the grid structure, we can simply store a single value - the index at which the grid cell start to contain particles in the sorted list.

So looking at Hash Cell 7 in the far right table, you can see that it has a value of 2.  This means that grid cell 7 contains particles starting at index 2 in the sorted list.  You can check to verify that this is indeed the case.  Then, the next Hash Cell that stores a value is number 10 with a value of 5.  You can then infer that Hash Cell 7 contains up to (but not including) index 5.  That way, we know what particles each grid cell contains.

Now, how does this help speed up nearest neighbor search?

Remember that before we simply looped over the particles to find our neighbors.  We're basically going to do the same thing here, but we've drastically reduced the number of particles we have to check.  If you carefully choose the grid cell size to be the same as the radius of your search, you only have to check the surrounding grid cells for potential neighbors.  By design any particle not within those cells will not pass the check of being less than the radius away from the current particle.

This image may be of use:
![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/hash_view.png)

This is a visualization of the hashing step described above.  Basically, particles in the same grid cell will be the same color in the image.  So, instead of searching through the entire point set for neighbors, you can just look in your cell (and those around yours).

###Performance

And now time for some performance evaluations.  First, we compared four different variations of the nearest neighbor search.  With/without the grid, and on the GPU/CPU.

First, here's a comparison of how particle number (and density) and neighbor amount affect performance.  Since that's a 3D data set, we got to make a 3D graph to show the results.  Videos can be found by clicking on the images below.

[![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/reg.jpg)](https://www.youtube.com/watch?v=ay8P2ykL2Xk&feature=youtu.be)

In the first video, you'll notice that the CPU brute force approach pops quite severely upwards as you increase both the neighbors and number of particles.  The other 3 stay much lower (you can't even see the GPU grid graph even).

[![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/zoom.jpg)](https://www.youtube.com/watch?v=HBxlWqiyqbc&feature=youtu.be)

We zoom in on the data to look at the differences between the other 3.  You'll notice that zoomed in you can see that the CPU Grid and GPU Brute Force Approaches do in fact get larger, while the GPU Grid stays flatter.  Another interesting fact is that the CPU Grid sometimes outpaces the GPU Brute Force.

First, let's take a look at the two methods on the GPU - using the grid and brute forcing it.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/gpu_comp.png)

One interesting thing to note is that the the brute force stays relatively flat, while the grid is growing.  This could be a problem when the number of neighbors you need approaches the size of your data set, or if your data set is highly dense.  Our data set was relatively dense, and another important factor is sheer number of particles.  This was on the order of 10,000 particles, which really is not that much.  The brute force approach breaks down with many more particles (as seen in the videos above).

One other important metric we wanted to compare is how much overhead the grid is costing us.  Clearly we're getting a net benefit from the grid structure, but we thought it would be interesting to quantify how much overhead we're paying for the speedup.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/gpu_brute_breakdown.png)

Clearly, in the brute force approach we're spending almost all of our time searching.  We don't need to set anything up (the memory management is done once, and thus is omitted from the comparison).

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/gpu_grid_breakdown.png)

However, in the grid approach, we're spending a lot of time not searching.  But that's a good thing!  The entire point of this structure is to reduce the search time.  The main costs here are the sorting step, and then the grid search.  The purple "other" represents mostly error checking.

Finally, we have to look at the memory overhead that comes with building this grid.  The cost of allocating the memory is amortized to 0, but we still need a sufficient amount of data to store the grid.  The extra amount of data is a function of the grid cell size, and will be discussed below.  The smaller the cell size, the greater the memory imprint.  Our grid is built around being unit sized, so your radius should be sufficiently small for most applications to prevent dampening or other loss of fine grained detail.

At simulations with lower particle count on the order of 1000, the memory imprint is significant in terms of percentage at lower radii:

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/memory_small_mesh.png)

However, it is important to note that this is still only on the order of 5 extra Mb in the worst case.

With higher particles counts, the extra memory becomes insignificant:

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/memory_big_mesh.png)

Overall, we're happy with grid - hopefully it will be of some use!

We put it to use in Jeremy's (kind of crappy) fluid sim.  It definitely sped things up!  With brute force search, the sim ran at just below 30 FPS, and with the grid, it ran at over 40 FPS.  So 1.3x increase in speed.  Not exactly revolutionary, but still pretty good.  See the video below.  (Unfortunately the only screen recording software we had access to was pretty bad, so you'll have to forgive the poor quality recording).

KD Tree
-----

Next up, we have a KD Tree acceleration structure for ray tracking.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/kd_tree_cover.png)

Similar to the uniform hash grid, this structure partitions the space into regions.  In this case, each cell of the tree contains a set of triangles of the mesh.  This is useful for ray tracking because when you want to intersect the mesh, you can use the tree structure to reduce the total number of intersections you have to compute.  Let's examine the bunny mesh without the KD Tree.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/mesh_render.bmp)

Now, the general case for intersecting this mesh using ray tracking is to simply loop over all of the triangles, finding the one that is the closes to the origin of the ray.  Pretty simple but has one glaring problem.  What about all of the pixels that have no chance of intersecting any triangle on the mesh?  That's the majority of the image!  We want to cull out those pixels and not even attempt ray tracking against the mesh.  The easiest way to do that is to first try intersecting the bounding box of the mesh.  If the ray doesn't hit that, there is no way that it could intersect a triangle on the mesh.

image

Now, you can think of a KD Tree as a set of bounding boxes of regions of the mesh.  Then we just arrange them in a fancy way that helps us accelerate this process.  The visualization below shows the number of ray-triangle intersections calculate at each pixel.  Green represents no intersections, while red indicates the max number of intersections.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/kd_intersections_brute.bmp)

Max number of intersections - 29808.  No acceleration structure at all - every pixel intersects every triangle.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/kd_intersections_bb.bmp)

Max number of intersections - 29808. Just a bounding box - every pixel that intersects the bounding box intersects every triangle.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/kd_intersections_2000.bmp)

Max number of intersections - 7147.  KD Tree with max 200 triangles / node.  You can vaguely see the global shape of the bunny

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/kd_intersections_200.bmp)

Max number of intersections - 1666.  KD Tree with max 200 triangles / node.  If you know it's supposed to be a bunny you can see it.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/kd_intersection_20.bmp)

Max number of intersections - 467.  KD Tree with max 20 triangles / node.  Very few intersection test computed.

So we're only intersecting when we need to, which is a great thing.  However you also have to look at the cost of building this structure, in terms of compute and memory.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/construction_v_traversal_graph.png)

As you can see, the construction time dwarfs an individual traversal time.  However, factoring in the large number of traversals needed to generate an image, it becomes clear that in general, you want to create an tree that can be traversed quickly, even it may cost a bit more to construct.

![](https://raw.githubusercontent.com/jeremynewlin/Accel/master/images/total_traversal_graph.png)