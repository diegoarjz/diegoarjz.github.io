---
layout: post
title:  "Creating a Simple Building with Pagoda"
date:   2020-04-28 12:43:00 +0100
categories: example pagoda
---


I decided to start my series of posts with an example of how we can create
a simple building using [pagoda](https://github.com/diegoarjz/pagoda).
Hopefully this will help understand how pagoda can be used to generate 3D
content.

Pagoda has a graph based approach to defining the rules with which we can generate our models. This means that nodes in this graph create, modify or otherwise perform any sort of operation on our objects and edges define the flow of objects between nodes.

As a heads up, pagoda does not yet have a GUI editor that would allow us to
visually create the rules to generate our models. Hopefully, I will be able to
get that going in the next release. For now, we can use the graph specification
language that pagoda provides.

So let's get started with our building which, in the end, will look something like the following image.

![Simple Building Final](/assets/images/posts/creating-a-simple-building/final.png)

The first step is to create a graph file and edit it with your favourite text editor. I creatively named this file `simple_building.pgd` and added the following:

```
building_volume = Operation(operation: "CreateBoxGeometry") {
    xSize: 30, ySize: 10, zSize: 20
}
building_volume_out = OutputInterface(interface: "out")

building_volume -> building_volume_out;
```

First, we define the building volume by creating a box with `(30, 10, 20)`
units along the `x`, `y`, and `z` directions respectively. To do this, we need
to create an `Operation` node with the `CreateBoxGeometry` operation. Objects
created with this operation are placed in the operation's `out` interface and,
therefore, we need an `OutputInterface` node to extract them and route them
downstream. By pagoda's convention, the `z` direction points upwards when we
are creating a new object. This means that our building will be 20 units tall.

If you now execute the graph you will see something similar to the following:

```
$ pagoda simple_building.pgd --execute
Info: Executing node 'building_volume'
Info: Executing node 'building_volume_out'
```

This will tell you which nodes are executing and in which order. It tells you that both nodes were executed, but where did the box end up? The answer is 'nowhere'. The box geometry was, indeed, created but it was then disposed of.

Since pagoda's philosophy is to have everything controlled by nodes and that each node or operation must have a very precise use, we need to explicitly tell it to do something with the geometry. To exemplify, we will export the geometry to an `obj` file by creating an `ExportGeometry` operation. We can do that with by adding the following snippet.

```
export_geometry_in = InputInterface(interface: "in")
export_geometry = Operation(operation: "ExportGeometry") {
    path: $< "out/geometry" + op.count + ".obj"; >$
}
export_geometry_in -> export_geometry;
```

The `ExportGeometry` operation exports all of the geometries that come in through its `in` input interface and writes it to the `obj` file specified in its `path` parameter. As such, we need to create an `InputInterface` node to route the objects to that interface.

The path parameter in the above snippet is a bit weird. This is because it is, in fact, using `pgscript` to specify the final path. Although not very important in this case, we needed a way to separate the various incoming objects into different files. That is why we are computing the final path based on the `op.count` exposed parameter.

All that is left to do is to connect the `out` output interface of the `CreateBoxGeometry` operation to the `in` input interface of the `ExportGeometry` operation. This is done like this:

```
building_volume_out -> export_geometry_in;
```

If you now run the graph, you can see the following output and a `geometry0.obj`inside the `out` folder. If that folder doesn't exist, pagoda creates it automatically.

```
$ pagoda simple_building.pgd --execute
Info: Executing node 'building_volume'
Info: Executing node 'building_volume_out'
Info: Executing node 'export_geometry_in'
Info: Executing node 'export_geometry'
```

As we move along, you can reuse the export geometry node we created to see how the geometry looks like at any point, by connecting output interface nodes to the `export_geometry_in` node. Because, in the end, we won't be using the geometry created by the `building_volume` node, you can delete disconnect the two nodes by deleting the line we added two snippets above.

The next step is to add a little bit of detail to the faces of the building volume but first we need to separate each face into its own object. The `ExtractFaces` operation does exactly this: for face of each object in its `in` interface creates an object in its `out` interface.

The question now becomes: _How can we distinguish the façades from the roof?_

This can be done with the `Router` node. This nodes evaluates predicates on the input objects and routes them to specific downstream nodes.

So let's see how we can do this. First the bit with the `ExtractFaces` operation:

```
building_faces_in = InputInterface(interface: "in")
building_faces = Operation(operation: "ExtractFaces")
building_faces_out = OutputInterface(interface: "out")
building_faces_in -> building_faces -> building_faces_out;

building_volume_out -> building_faces_in;
```

And now the `Router` node:

```
building_face_router = Router() {
    facade_in: "side",
    roof_in: "up"
}

building_faces_out -> building_face_router;
```

In the snippet above, the execution parameter names (facade and roof) correspond to downstream nodes (we'll get to that next) and the "side" and "up" are predicate names that pagoda provides. In this case, the node is routing the side faces to the `facade_in` downstream node and the up face to the `roof_in` downstream node.

Let's get to adding some detail to the roof. We will introduce the `FaceOffset` operation which takes faces in the incoming objects and creates offset faces at a given distance from the original face's edge.

```
roof_in = InputInterface(interface: "in")
roof = Operation(operation: "FaceOffset") {
    amount: 4
}
roof_inner = OutputInterface(interface: "inner")
roof_outer = OutputInterface(interface: "outer")
roof_in -> roof -> roof_inner;
roof -> roof_outer;

building_face_router -> roof_in;
```

The `FaceOffset` operation has two output interfaces allowing to distinguish
between interior and border objects. As you can see in the snippet above, these
interfaces are named _inner_ and _outer_ respectively.

We will achieve the last detail in the roof by extruding the inner face
generated by the above operation. Conveniently, there is an `ExtrudeGeometry`
operation. Let's add it to the graph file.

```
roof_extrusion_in = InputInterface(interface: "in")
roof_extrusion = Operation(operation: "ExtrudeGeometry") {
    extrusion_amount: 1
}
roof_extrusion_out = OutputInterface(interface: "out")
roof_extrusion_in -> roof_extrusion -> roof_extrusion_out;

roof_inner -> roof_extrusion_in;
```

Next, we can divide the façades into floors and then into windows. This can be
done with two chained `RepeatSplit` operations, alternating its axis. First, we
split along the _y_ axis and then the _x_ axis. This will give us a grid like
structure in the façade.

Remember earlier in this post when I mentioned that the _z_ direction pointed
upwards? So why are we splitting the façades along the _y_ direction first? The
reason is that every object that is created in pagoda has a _scope_ which is,
basically, an oriented bounding box. Implicitly, this defines a frame of
coordinates for each object and in the case of our façades, the _x_ axis moves
left to right along the bottom edge, the _y_ axis moves bottom to top along the
left edge. Consequentially, the _z_ axis points outwards.

```
facade_in = InputInterface(interface: "in")
facade = Operation(operation: "RepeatSplit") {
    axis: "y",
    size: 1.5,
    adjust: "true"
}
facade_out = OutputInterface(interface: "out")
facade_in -> facade -> facade_out;
building_face_router -> facade_in;

floor_in = InputInterface(interface: "in")
floor = Operation(operation: "RepeatSplit") {
    axis: "x",
    size: 1,
    adjust: "true"
}
floor_out = OutputInterface(interface: "out")
floor_in -> floor -> floor_out;

facade_out -> floor_in;
```

The final bit is to export the geometries we're interested in. We can use the `ExportGeometry` operation we defined above to which we connect the `floor_out` `roof_extrusion_out` and `roof_outer` nodes as follows.

```
floor_out -> export_geometry_in;
roof_extrusion_out -> export_geometry_in;
roof_outer -> export_geometry_in;
```

Your `simple_building.pgd` file should now look something like this:

```
building_volume = Operation(operation: "CreateBoxGeometry") {
    xSize: 30, ySize: 10, zSize: 20
}
building_volume_out = OutputInterface(interface: "out")
building_volume -> building_volume_out;

export_geometry_in = InputInterface(interface: "in")
export_geometry = Operation(operation: "ExportGeometry") {
    path: $< "out/geometry" + op.count + ".obj"; >$
}
export_geometry_in -> export_geometry;

building_faces_in = InputInterface(interface: "in")
building_faces = Operation(operation: "ExtractFaces")
building_faces_out = OutputInterface(interface: "out")
building_faces_in -> building_faces -> building_faces_out;

building_volume_out -> building_faces_in;

building_face_router = Router() {
    facade_in: "side",
    roof_in: "up"
}
building_faces_out -> building_face_router;

roof_in = InputInterface(interface: "in")
roof = Operation(operation: "FaceOffset") {
    amount: 4
}
roof_inner = OutputInterface(interface: "inner")
roof_outer = OutputInterface(interface: "outer")
roof_in -> roof -> roof_inner;
roof -> roof_outer;

building_face_router -> roof_in;

roof_extrusion_in = InputInterface(interface: "in")
roof_extrusion = Operation(operation: "ExtrudeGeometry") {
    extrusion_amount: 1
}
roof_extrusion_out = OutputInterface(interface: "out")
roof_extrusion_in -> roof_extrusion -> roof_extrusion_out;

roof_inner -> roof_extrusion_in;

facade_in = InputInterface(interface: "in")
facade = Operation(operation: "RepeatSplit") {
    axis: "y",
    size: 1.5,
    adjust: "true"
}
facade_out = OutputInterface(interface: "out")
facade_in -> facade -> facade_out;
building_face_router -> facade_in;

floor_in = InputInterface(interface: "in")
floor = Operation(operation: "RepeatSplit") {
    axis: "x",
    size: 1,
    adjust: "true"
}
floor_out = OutputInterface(interface: "out")
floor_in -> floor -> floor_out;

facade_out -> floor_in;

floor_out -> export_geometry_in;
roof_extrusion_out -> export_geometry_in;
roof_outer -> export_geometry_in;
```

You can now run the graph file to generate the whole building. You will see the as each node is executed and, in the end, all files generated by pagoda will be output to the `out` directory.

```
$ pagoda final.pgd --execute
Info: Executing node 'building_volume'
Info: Executing node 'building_volume_out'
Info: Executing node 'building_faces_in'
Info: Executing node 'building_faces'
Info: Executing node 'building_faces_out'
Info: Executing node 'building_face_router'
Info: Executing node 'roof_in'
Info: Executing node 'facade_in'
Info: Executing node 'roof'
Info: Executing node 'facade'
Info: Executing node 'roof_inner'
Info: Executing node 'roof_outer'
Info: Executing node 'facade_out'
Info: Executing node 'roof_extrusion_in'
Info: Executing node 'floor_in'
Info: Executing node 'roof_extrusion'
Info: Executing node 'floor'
Info: Executing node 'roof_extrusion_out'
Info: Executing node 'floor_out'
Info: Executing node 'export_geometry_in'
Info: Executing node 'export_geometry'
```

You're probably wondering why there are so many `obj` files in the `out` directory. The reason is that, currently, pagoda creates a new geometry for each object that it creates. In the future, we might see some reuse of geometries between different objects and, instead of ending up with hundreds of files, we might get a single geometry file with the entire building.

For the time being, however, we're stuck with importing all these files to (for example) blender to get a nice render like the one at the top of the post.

I hope you enjoyed reading this post and, if you have any question, suggestion or otherwise just want to get in touch, feel free to send me an email or reach out via Twitter.
