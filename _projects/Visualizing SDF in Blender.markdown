---
layout: distill
title: Visualizing SDF in Blender
description: Colorcoding distance to the object with Blender Geometry Nodes 
img: /assets/img/sdf_torus/0030.png
importance: 2
category: blender
date: 2024-01-27

# bibliography: 2021-vectorgan.bib
toc:
  - name: Setting up a scene
    #   - name: Example Child Subsection 1
    #   - name: Example Child Subsection 2
  - name: Setting up color attribute
  - name: Geometry node
  - name: “Signed” distance value
  - name: Animation with keyframes

# authors:
#   - name: Ivan Puhachov
#     url: "https://puhachov.xyz"
#     affiliations:
#       name: UdeM, Canada
---
TLDR: jump to [Geometry Node](#geometry-node)

Visualizing SDF in Blender. Normally, one would use KBD-tree to process input mesh and compute SDF value for any query point. We don’t want that, we want something interactive, simple, and in directly Blender - come and meet the Geometry Nodes!

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/sdf_torus/0030.png" data-zoomable>
    </div>
</div>
<div class="caption">
    Example visualization with Torus and Plane. Scroll down for animation!
</div>

We will use Vertex Color Attribute to store color properties for each vertex in the cross-section plane. And Color Attribute is computed on-the-go with Geometry Nodes.


## Setting up a scene
We want to view SDF values for `Torus` on `Plane` cross-section. 
You can use any “cross-section” surface really, as long as it has enough vertices so it looks nice (mesh colors will be interpolated by per-vertex color attribute). For default `Plane` object in Blender you need to subdivide it: `Tab`  to get to edit mode, then `F3`  to search and use `mesh.subdivide`  operation. 
<div class="l-gutter">
  {% include figure.liquid path="/assets/img/sdf_torus/image.png" class="img-fluid rounded z-depth-1" zoomable=true %}
</div>


## Setting up color attribute 

In `Object Data` menu create new `Color Attribute` (I name it `new_color_attribute` here). Then make sure your Material BSDF takes this color attribute as mesh vertex color
<div class="l-gutter">
  {% include figure.liquid path="/assets/img/sdf_torus/image2.png" class="img-fluid rounded z-depth-1" zoomable=true %}
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/sdf_torus/image3.png" data-zoomable>
    </div>
</div>

## Geometry node

Now we fill values of the color attribute using Geometry Node. In `Modifiers` menu create a new Geometry Node and fill it with these nodes:

<div class="l-page">
  {% include figure.liquid path="/assets/img/sdf_torus/image4.png" class="img-fluid rounded z-depth-1" zoomable=true %}
</div> 

Geometry Node design is pretty self-explanatory: for each vertex `Position`  we compute distance with `Geometry Proximity` node, then map this distance value to `Color Ramp` with `Map Range`, and write the color to `new_color_attribute`.\\
Don’t forget to fill the `Object` input field in Modifier info as shown here:  
<div class="l-gutter">
  {% include figure.liquid path="/assets/img/sdf_torus/image5.png" class="img-fluid rounded z-depth-1" zoomable=true %}
</div>

We now have something like this:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/sdf_torus/sdf1.png">
    </div>
</div>

  
## “Signed” distance value

Well, we don’t have “signed” values, do we? I haven’t found a way to fix this with Geometry Nodes, but we can do it much simpler: just copy the node and create the same one for the “inside” portion of visualization (negative values) and change the colors.

<div class="l-page">
  {% include figure.liquid path="/assets/img/sdf_torus/image6.png" class="img-fluid rounded z-depth-1" zoomable=true %}
</div> 

We then use boolean operations to get the “inner” and “outer” part. Duplicate existing plane and use `Intersect` boolean modifier `Intersect`. Do the same for the first plane, but taking the `Difference` instead. 

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
      <figure class="mx-auto" style="max-width: 300px;">
      <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/sdf_torus/image7.png" data-zoomable>
      </figure>
    </div>
</div>

Tip: select both planes and hit `Ctrl + P`  to build a hierarchy which preserves transformations (`Keep Transform`). This will help with animating the plane later.

<div class="l-gutter">
  {% include figure.liquid path="/assets/img/sdf_torus/image8.png" class="img-fluid rounded z-depth-1" zoomable=true %}
</div>

Now you get something like the top figure. You can play with color ramp, lights and shade to make it better. Blender Freestyle overlay may help with volume, and Toon Shader is nice to flatten out the shape.
<div class="l-gutter">
  {% include figure.liquid path="/assets/img/sdf_torus/sdf2.png" class="img-fluid rounded z-depth-1" zoomable=true %}
</div>

## Animation with keyframes
We can put keyframes on plane position and animate its movement. Once rendered, compose a video file from png images with:

`ffmpeg -framerate 12 -i anim_sdf/%4d.png -c:v libx264 output.mp4`

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include video.liquid path="/assets/img/sdf_torus/sdf.mp4" class="img-fluid rounded z-depth-2" controls=true autoplay=false %}
    </div>
</div>