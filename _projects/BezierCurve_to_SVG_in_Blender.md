---
layout: distill
title: Render Bezier to SVG in Blender
description: Rendering 3D vector curves to 2D vector curves (polylines) with visibility in Blender
img: assets/img/svg_blender_rendering/result_torus.png
importance: 2
category: blender
date: 2024-01-27

# bibliography: 2021-vectorgan.bib
toc:
  - name: Python Script
    #   - name: Example Child Subsection 1
    #   - name: Example Child Subsection 2
  - name: Add pip package to Blender Python
#   - name: Geometry node
#   - name: “Signed” distance value
#   - name: Animation with keyframes

# authors:
#   - name: Ivan Puhachov
#     url: "https://puhachov.xyz"
#     affiliations:
#       name: UdeM, Canada
---
Setup: you have a BezierCurve in blender, and you would like to render it to svg file.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/img/svg_blender_rendering/blender_screen.png" class="img-fluid rounded z-depth-1 no-shadow" zoomable=true%}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/img/svg_blender_rendering/result_torus.png" class="img-fluid rounded z-depth-1 no-shadow" zoomable=true%}
    </div>
</div>

SVG file: [winding.svg](/assets/img/svg_blender_rendering/winding.svg)

You can render curves with **Blender Add-on with Bézier Utility Operations** ([https://github.com/Shriinivas/blenderbezierutils](https://github.com/Shriinivas/blenderbezierutils)) but it does not take into account **curve visibility** (curve is still there even though it is occluded by an object in the scene). See the inset here.
<div class="l-gutter">
  {% include figure.liquid path="/assets/img/svg_blender_rendering/result_torus_bad.png" class="img-fluid rounded z-depth-1" zoomable=true %}
</div>

SVG file: [winding_wrong.svg](/assets/img/svg_blender_rendering/winding_wrong.svg)

## Python Script

TLDR: sample points on the curve and check if this point is visible from camera or it is occluded by an object in your scene – see `ray_cast` function. Then compose polylines from visible points.

> **Limitation**: the result will be a collection of polylines, not bezier curves. It seems possible to do it properly by projecting bezier curve on screen space and splitting it into visible and invisible parts using same logic as here.

**Warning**: this is a demo script that only takes one curve. You might want to update it to render multiple curves to one SVG file  

```python
import bpy
import bpy_extras
import svgwrite

curve = bpy.data.objects.get("myCurve.003")  # name of curve to render. make this script a loop if you want to iterate over many curves
camera = bpy.data.objects.get("Camera")

# Get the active scene
scene = bpy.context.scene

# Get the dependency graph for the scene
depsgraph = bpy.context.evaluated_depsgraph_get()

list_of_lines = []
current_line = []

# here we form a list of 3d points to check for visibility and build polylines
# assuming that Bezier curve is densely subdivided, we only look a the vertices
points_to_check = list(enumerate(curve.data.splines[0].bezier_points))
# if curve data contains multiple splines, iterate over curve.data.splines to form points_to_check

# when cyclic_u we need to double-check the starting point
if curve.data.splines[0].use_cyclic_u:
    points_to_check.append(points_to_check[0])

for (i,bp) in points_to_check:
    # https://docs.blender.org/api/current/bpy.types.Scene.html#bpy.types.Scene.ray_cast
    point = bp.co
    direction = (point - camera.location).normalized()
#    print(f"\n===={i}")
#    print(f"Point: {point}")
#    print("Direction:", direction)
    is_intersecting_solids, location, normal, index, object, matrix = bpy.context.scene.ray_cast(
        depsgraph,
        camera.location,
        direction,
    )
    # ray_cast will not compute intersection with Bezier curve, so is_intersecting_solids is True when ray falls on some solid object in your scene
    # let's check if the interection happens after we see this point on a curve
    # if is_intersecting_solids is False we don't need to compute it
    is_intersecting_after = False
    if is_intersecting_solids:
        distance_curve = (point - camera.location).length
        distance_object = (location - camera.location).length
#        print(f"distance curve: {distance_curve}")
#        print(f"distance object: {distance_object}")
        is_intersecting_after = distance_object > distance_curve
    
    point_is_visible = (not is_intersecting_solids) or is_intersecting_after
    
    # point is not occluded by a shape. now we find its pixel coordinates and don't plot it if it is outside the frame
    point_is_in_frame = False
    if point_is_visible:
        # check that it is in frame using normalized device coordinates (NDC)
        # https://docs.blender.org/api/current/bpy_extras.object_utils.html#bpy_extras.object_utils.world_to_camera_view
        ndc_coords = bpy_extras.object_utils.world_to_camera_view(
            scene, camera, point
        )
#        print(ndc_coords)
        if (0<= ndc_coords.x <= 1) and (0<= ndc_coords.y <= 1) and (ndc_coords.z >= 0):
            point_is_in_frame = True
            # calculate pixel coordinates in camera
            render_scale = scene.render.resolution_percentage / 100
            point2d = [ndc_coords.x * scene.render.resolution_x * render_scale, (1 - ndc_coords.y) * scene.render.resolution_y * render_scale]
            current_line.append(point2d)
#            print("point 2d:", point2d)
    if point_is_in_frame:
        continue
    else:
        if len(current_line) > 1:
            list_of_lines.append(current_line)
        current_line = [] 
#    break

if len(current_line) > 1:
    list_of_lines.append(current_line)
    current_line = [] 

print(f"\n---> found {len(list_of_lines)} lines")
#print(list_of_lines)


# utility function to output svg file
def write_lines_tosvg(
    lines_list,
    filename="lines2.svg",
    width=scene.render.resolution_x,
    height=scene.render.resolution_y,
):
    dwg = svgwrite.Drawing(filename, profile='tiny')
    dwg['width'] = '{}px'.format(width)
    dwg['height'] = '{}px'.format(height)
    for line in lines_list:
        polyline = dwg.polyline(line, stroke='black', fill='none')
        dwg.add(polyline)
    dwg.save()


write_lines_tosvg(list_of_lines)
```

  

### Add pip package to Blender Python

In macos:   

```bash
./Applications/Blender.app/Contents/Resources/4.0/python/bin/python3.10 -m pip install svgwrite
```