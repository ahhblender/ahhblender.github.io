---
title: How to Use Vertex Color Layers
date: 2023-08-05 10:59:45 -0700
categories: [Blender Python API]
tags: [blender, python]
---

In the **Object Data Properties** there's a section called *Color Attributes* that allow you to create color layers to colorize components of a mesh. There are various debugging tools where this kind of functionality is helpful, such as colorizing a mesh based on the number of bone influences each vertex has.

![color_attributes_ui](/docs/assets/images/color_attributes_ui.png){: w="400" }

<br>
I was feeling pretty good. I sorted out all the functions I'd need to work with them. Then as I was making this post I saw this:

![vertex_colors_deprecated](/docs/assets/images/vertex_colors_deprecated.png){: w="700" }
![ugh](/docs/assets/images/ugh.png){: w="150" }

Well, anyways, if you wanted to know how to use Python to work with the **DEPRECATED** vertex colors, I got some news for you. Keep on reading and you too will know of the old ways of vertex colors.

<br>
Here's an example of how to create a vertex color layer.

```python
import bpy
from typing import Collection, Iterable, Tuple


def create_vertex_color_layer(mesh_obj: bpy.types.Object,
                              vertex_indices: Iterable[int],
                              color: Tuple[float, float, float],
                              color_layer_name: str,
                              set_active: bool = True,
                              set_viewport: bool = True) -> None:
    if color_layer_name not in mesh_obj.data.vertex_colors:
        mesh_obj.data.vertex_colors.new(name=color_layer_name)

    vertex_color_layer = mesh_obj.data.vertex_colors[color_layer_name]

    for poly in mesh_obj.data.polygons:
        for loop_index in poly.loop_indices:
            vertex_index = mesh_obj.data.loops[loop_index].vertex_index
            if vertex_index in vertex_indices:
                vertex_color_layer.data[loop_index].color = (*color, 1.0)

    if set_active:
        set_active_vertex_color_layer(mesh_obj, color_layer_name)
    if set_viewport:
        set_viewport_color_type('VERTEX')


if __name__ == "__main__":
    suzanne_mesh = bpy.data.objects.get('Suzanne')

    vertex_indices = range(len(suzanne_mesh.data.vertices))
    color_layer_name = 'my_layer'

    create_vertex_color_layer(suzanne_mesh, vertex_indices, (0, 0, 1), color_layer_name)
```
<br>
![color_attributes_ui](/docs/assets/images/viewport_color_attribute.png){: w="400" .right }
Here are the functions that are called from the two flag arguments. One sets the active vertex color layer. This is helpful for when there's more than one vertex color layer and you want the one you just created to show up.

The second function lets you set the color type within the viewport. **You'll need to switch the color type to Attribute or you won't see the vertex colors.**

As far as I know, here are the strings you can set the viewport color to: *'MATERIAL', 'TEXTURE', 'VERTEX', 'VERTEX_ALPHA', 'SINGLE', 'OBJECT', 'RANDOM'*.

That was all I had to say about those, but then there was this awkward space here that I had to fill up with something. So yeah, this is all I could come up with.

<br>

```python
def set_active_vertex_color_layer(mesh_obj: bpy.types.Object, color_layer_name: str) -> None:
    if color_layer_name in mesh_obj.data.vertex_colors:
        color_layer_index = mesh_obj.data.vertex_colors.find(color_layer_name)
        mesh_obj.data.vertex_colors.active_index = color_layer_index
        
        
def set_viewport_color_type(color_type: str) -> None:
    for area in bpy.context.screen.areas:
        if area.type == 'VIEW_3D':
            for space in area.spaces:
                if space.type == 'VIEW_3D':
                    space.shading.color_type = color_type
```

<br>
Now that we can *create* them, we'll also need to *remove* them too, and what do you know, here's a function to do just that.

```python
def remove_vertex_color_layer(mesh_obj: bpy.types.Object, color_attribute_name: str, refresh_viewport: bool = True) -> None:
    if color_attribute_name in mesh_obj.data.vertex_colors:
        color_layer = mesh_obj.data.vertex_colors[color_attribute_name]
        mesh_obj.data.vertex_colors.remove(color_layer)

        if refresh_viewport:
            # hack to update the viewport mesh colors
            mesh_obj.hide_viewport = True
            bpy.ops.wm.redraw_timer(type='DRAW_WIN_SWAP', iterations=1)
            mesh_obj.hide_viewport = False


if __name__ == "__main__":
    suzanne_mesh = bpy.data.objects.get('Suzanne')
    color_layer_name = 'my_layer'

    remove_vertex_color_layer(suzanne_mesh, color_layer_name)
```
<br>
Now, about that hack to update the viewport after removing the color layer. Normally, if you remove the vertex color layer you won't see it update in the viewport without disabling the mesh in the viewport *(clicking the TV icon in the Outliner)* and re-enabling it, but I wanted the update to be instantaneous. After trying various things I found that this was the one thing that worked.

Use **hide_viewport** specifically to hide the mesh, call the redraw function, and then unhide the mesh. And voila! The colors are immediate removed. One thing to watch out for is if you're doing this to many meshes at once. Calling the redraw function over and over will slow down your script.

I wanted to keep the functionality of remove_vertex_color_layer() that could refresh the viewport when working with a single mesh, but when working with many meshes I didn't want to keep calling the redraw function. Here's how I got around it.

First of all, we gotta hide all the meshes. Then we'll loop through them and keep track of which index of the list we're on. When we're on the last element, we'll set refresh_viewport to True and trigger the redraw. Then we'll unhide all the meshes.

```python
def remove_vertex_color_layers_fancy_pants(mesh_objs: Collection[bpy.types.Object], color_attribute_name: str, refresh_viewport: bool = True) -> None:
    if refresh_viewport:
        for mesh_obj in mesh_objs:
            mesh_obj.hide_viewport = True
        meshes_total = len(mesh_objs) - 1

    for index, mesh_obj in enumerate(mesh_objs):
        if refresh_viewport:
            is_last_element = index == meshes_total
            remove_vertex_color_layer(mesh_obj, color_attribute_name, refresh_viewport=is_last_element)
        else:
            remove_vertex_color_layer(mesh_obj, color_attribute_name, refresh_viewport=False)

    if refresh_viewport:
        for mesh_obj in mesh_objs:
            mesh_obj.hide_viewport = False
```
<br>
Then I realized, the code to remove the color layers is so small, why not just make the whole thing a lot simpler? Here's is the final version:

```python
def remove_vertex_color_layers(mesh_objs: Iterable[bpy.types.Object], color_attribute_name: str, refresh_viewport: bool = True) -> None:
    if refresh_viewport:
        for mesh_obj in mesh_objs:
            mesh_obj.hide_viewport = True

    for mesh_obj in mesh_objs:
        if color_attribute_name in mesh_obj.data.vertex_colors:
            color_layer = mesh_obj.data.vertex_colors[color_attribute_name]
            mesh_obj.data.vertex_colors.remove(color_layer)

    if refresh_viewport:
        bpy.ops.wm.redraw_timer(type='DRAW_WIN_SWAP', iterations=1)
        for mesh_obj in mesh_objs:
            mesh_obj.hide_viewport = False


if __name__ == "__main__":
    suzanne_mesh = bpy.data.objects.get('Suzanne')
    suzanne2_mesh = bpy.data.objects.get('Suzanne2')

    vertex_indices = set(range(len(suzanne_mesh.data.vertices)))
    color_layer_name = 'my_layer'
    
    remove_vertex_color_layers((suzanne_mesh, suzanne2_mesh), color_layer_name)
```

<br>
I hope you enjoy your new **DEPRECATED** knowledge of vertex colors. Now how do we use the newer, non-deprecated style with color attributes? Hmm.