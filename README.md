///////////////////////////////////////////////////////
//             Heatmap Service Library               //
// Written by: Pedro Engana (http://pedroengana.com) //
///////////////////////////////////////////////////////

The HeatmapService library provides a robust and easy to use tool to log player activity spatially organized over 2d space.
It can log any counters on a coordinate basis to measure activity, such as deaths, gold collected, materials gathered, etc...

The Heatmap can be queried for it's stored values, from the smallest unit of space to the entirety of the record.
It can also serialize itself to a char* buffer and can later restored from the same buffer.
It comes with a simple console application for testing purposes.


///////////////////////////////////
////// Running the lib tests //////
///////////////////////////////////

1. Open the .sln file (HeatmapServiceLib.sln) with a Visual Studio that supports Format Version 12.00
2. Build or Rebuild HeatmapServiceTests (this will also build the lib itself)
3. Run, and watch the automated tests results

///////////////////////////////
////// Solution Overview //////
///////////////////////////////

The Heatmap Service Lib is constituted of 3 main components:

HeatmapService ---> Heatmap ---> CounterMap

- Heatmap Service: A container for the inner functionality of the library. 
It occludes implementation details from the library header, and provides the library API.

- Heatmap: The actual internal implementation of the Heatmap, 
It contains different CounterMaps for each counter (deaths, gold caught, etc...) it's responsible for incrementing the counters, querying and serializing itself and all it's maps. It also receives a spatial resolution at the start, defining the size of a single unit of space in the heatmap (10m, 10inches, etc...), all coordinates logged to the map are transformed according to the spatial resolution.

- CounterMap: Is the low level matrix of accumulators.
It implements a dynamically increasing matrix that can grow on Y independently for each column, and always provides access times of O(1) complexity. My greatest concern in building this class was on speed of access, over memory. I figured that, regardless of my wishes, with use, the map will eventually have a counter for every point in it. Heatmaps after all, are not even useful if they have sparse data, so I decided that hash-maps for coordinates or a binary search tree would eventually be doing more ill than good.

This however doesn't mean I disregarded memory all together. Instead I designed a solution that would retain O(1) but with an algorithm that avoids allocating memory as much as it can.
The counter map instances a matrix of values, but not the whole matrix. Each column can grow independently of it's peers. For example, if I ask to increment the counter at {0,0} and then at {100,100} I won't end up with 100*100 positions allocated, it will instead only grow the column at index 100, resulting in an allocation of only 100 positions for that column (not exactly... continue reading). This allows me to keep the memory footprint of the heatmap as low as possible, for as long as possible.

This however is not the whole truth. Allocation operations are expensive, and the map shouldn't be doing them all the time, if coordinates were logged from 0,0 outwards it would constantly re-allocate itself. Instead of allocating exactly the memory needed, the column is always doubled in size. With this I hope to quickly reach a size that can contain the map. I figured that even if I don't know the initial size, if I quickly grow the map, I will eventually stabilize at a size that can contain all possible future registries. The source map where the logs are coming from is very unlikely to change size overtime. This has a bit of a memory hit, as more likely than not more memory will be allocated than actually needed, but I felt it was justified.


///////////////////
////// Usage //////
///////////////////

- Registering values:
To register values to the heatmap, the Increment methods should be called. A coordinate should be passed (any two double values, x and y. The heatmap supports both negative coordinates as well as fractional) as well as the key for the counter to register to, counter keys must be references to const std::strings.

- Querying values:
Querying values is similar to registering them, a coordinate and a counter key need to be provided.
In case the given coordinates, or the counter key, were never logged before the value returned is 0.
Queries can also be made in an area of the map, for this a rectangle must be provided, represented by lowest point and the highest point. In area queries, the data structure HeatmapData is returned, containing a matrix of the data in the area, as well as information about the data retrieved.

- Serializing the Heatmap
The Heatmap can serialize itself to a char array, and later recovered from the same data. The library uses boost for serialization purposes, but writes the stream to the char array ensuring any application that uses the lib, doesn't need to use boost serialization itself. The required boost libraries are, of course, bundled with this project to ensure it works properly.


/////////////////////////////////////////////////////
////// Discussion on Assumptions and Decisions //////
/////////////////////////////////////////////////////
In this section I'll document and discuss some of my design decisions given the high level requirements that were presented to me.

- The Pimpl Idiom as a way to occlude private library info:
This one wasn't a hard decision, but right at the start I decided that If I was building a library, then I needed to approach it as a separate project. As self-contained as possible and as "black-box" as possible, to prevent any kind of wrong use. No private implementation should be shown. For this I used the private implementation idiom (PIMPL).

- Counter Keys as strings: 
Despite some initial concerns about having a "stringly typed" library, I ended up deciding to use std::strings as the way for identifying keys due to their versatility. This way, any counter imaginable can be registered by name, without having to change any file connected to the application to add new counters. It also means that it can be used in different applications and the serialized data will always be valid. All this could also be achieved with an int of course, but it wouldn't be as expressive, so I decided to use strings, also assuming that in normal applications of the lib, no more than 20 or 30 different counters are likely to be used, so the performance and memory hits wouldn't be too significant.

- Coordinates as double values: 
Even though it wasn't defined in the requirements, I decided that the heatmap, in order to be as versatile as possible, should be easy to use for any 2d coordinates, without requiring transformations. As such, I needed to support fractional values as well as negatives so that coordinates such as {1, 5}, {200, 400}, and even {124.4, -20232.521} can be used interchangeably and correctly logged. As can be seen in the code, I designed the library in a way that supporting these coordinates has little performance or memory repercussions (aside from having to pass a few more bytes on function calls).

- KeyMap Dictionary on HeatmapPrivate.h and Heatmap.cpp
This structure is a very simple array of key values to index counter keys to their respective counter maps.
This structure isn't the most speedy to access (O(n)), but it's simple, memory efficient and pretty explicit. I decided to implement it as such since I felt I didn't need the most well performing structure to index keys. Following the same assumption I described above, in a regular use of the lib no more than 20 or 30 different counters are likely to be used, so the performance hits won't be too significant.

- Use of Raw Pointers VS smart pointers
In modern C++, using smart pointers is very strongly encouraged, and in regular situations, if the performance hit wasn't significant, I would definitely use them. They are excellent at preventing mistakes. Here however, I decided to go with raw pointers. The reasoning for it is simple. The requirement that I don't use STL gives me hints that this exercise was to also to evaluate my skills with memory, so this is a way for me to "show off" that I can manually deal with memory if necessary.

- Boost Serialization
Serialization can be quite tricky, and there's already a bunch of very good libraries that implement efficient and error-free serialization functionalities. Boost is the obvious choice for this, as it's libraries are widely used and are even in the process of being introduced into the c++ standard. No need to re-invent wheels, when there's a perfectly good one ready for use. I did however, still ask Thomas Abrams from HR If I could in fact use it, and the response was positive.

- Thread Safety
I pondered for a while If I should make the Heatmap thread safe or not. In the end, I decided against it. Even though it would be a nice to have feature, it shouldn't be it's default responsibility to make itself thread safe, instead, the software that uses the lib should take care not to do parallel calls. A thread safe extension to the heatmap could perhaps make sense.


////////////////////////////////////
////// Future Work & What-ifs //////
////////////////////////////////////
Stuff I would develop for this heatmap in the future, but didn't make sense to develop now.
Also know as: "where feature creep goes to die".

- Proper Unit Testing:
Since I just wanted to quickly test the implementation and as tests were not required by the problem, I implemented a simple set of tests in the test console app.
With more time, I would build proper unit tests for each of the modules of the library

- BMP or PNG printing:
Printing the heatmap data to an image file would be pretty cool!

- Merging Heatmaps
Merging two heatmaps together, or additively de-serializing into another could be potentially useful, but a bit complex if they have different spatial resolutions.

- Writing it's own serialization files parallel to serializing to char* functionality
.heatmap files for easy saving and loading, not requiring the software in charge to do it would definitely be useful, but outside the scope of the exercise.

- Using STL
If I didn't have the explicit requirement that I shouldn't use STL, I would have used it, if performance and memory wasn't too much of a bottleneck. The containers in the library are much safer to use and easier to read in code.
