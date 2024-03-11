GPU-accelerated rendering PBF files onto an interactive canvas.

![Rendering of Melbourne CBD](/images/melbourne.webp)

Melbourne's CBD, minus the Yarra river which is now a void.

Legend:

- Yellow: Road
- Blue: Rail (tram and train) line
- Green: Park land
- Grey: Building footprint
- Orange dots: Footpath

# Overview

OpenGL 4.10 (thanks Apple) is used as the graphics API for rendering these files. To achieve this,
I created a relatively light-weight wrapper over the raw OpenGL calls (provided by the `gl` crate)
to assist with the creation of the shader programs and associated buffers, uniforms, etc. OpenGL
was originally chosen due to my familiarity with it, however I'm interested in porting this to WGPU
at some point for better compatibility.

The PBF files are loaded and processed on the CPU, with the ways (OpenStreetMap terminology for
what's effectively a collection of nodes representing _something_, such as a building or a road)
filtered out to only include objects of interest (namely roads, parks, buildings, and railways,
which occurs in `/src/osm/way.rs`). These objects of interest then have bespoke rendering rules
(defined per object in `/src/objects/*.rs`) which effectively boil down to drawing either a line
and/or some fill. These rendering instructions and node data is all packaged up and sent over to
the GPU which will generate any required geometry (such as each of the rectangles for a dashed line
or rounded vorners). This technique offers a number of benefits, including minimising the amount of
data that needs to be sent to the GPU and the time that the CPU spends processing the data. It also
decouples the interpretation and style of the map objects from the underlying graphics API,
(theoretically) making it simpler to migrate at some point.

The renderer has a perspective camera, and is looking down at a 2D plane in a 3D environment. This
allows (fairly trivially) for extrusions and models to be included on the map, such as extruding
the building footprints, or including models for traffic and other dynamic elements. This could
even extend to plugging 'real world' data sources in (such as traffic or public transport
timetables) to view them in real time on the map. In order to handle mouse interactions with the
map (such as clicking and panning), some slightly curly maths was required to project the mouse
from camera space into world space, but the resulting effect is very plesant to use!

# Resources

 - [Overpass](https://overpass-turbo.eu/) - Query and select data, and export it to a number of different files
 - [OSM Export](https://www.openstreetmap.org/export#map=19/-37.79574/144.92911) - Easily grab and export a bounding box on a map

# OSM Queries

## Get residential roads within the bounding box

```osm
[timeout:25][bbox:-37.796223, 144.928243, -37.796100, 144.928732];

(
  way["highway"="residential"];
);

out body;
>;
out skel qt;
```

# Other Notes

## Getting data into PBF format

1. Download desired data from one of the links above (export raw data from Overpass API)
2. Download and compile `osmconvert`: `http get http://m.m.i24.cc/osmconvert.c | cc -x c - -lz -O3 -o osmconvert`
3. Run the following command to perform the conversion: `cat [downloaded file] | ./osmconvert - --out-pbf -o=[PBF output file]`

