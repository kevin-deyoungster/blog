---
toc: true
layout: post
description: An end-to-end recreation of GhanaPostGPS using entirely open-source technologies
categories: [GhanaPost, Digital Address, Geospatial, Python, React, GIS]
title: Recreating Ghana's Digital Address System
---

![](https://i.imgur.com/L2aIeHC.png)
*Screenshot of the frontend for this project (GhanaPostGPS clone). The map on the right shows the location marker and its cell (red polygon). The sidebar on the left shows the address details. **You can access the [demo](https://cutt.ly/ghanapostclone2) here**.*

## Introduction

In 2017, the Ghana government introduced a digital addressing system
called GhanaPostGPS, which effectively provides an address for "every
location and place in the country". GhanaPostGPS was built with the
intent of enhancing the delivery of social services (health services
like ambulance, fire depart, law enforcement, electricity provision and
a revamped postal system)[^1]. The project, of course, targeting a
nation-wide scale, was estimated to cost \$2.5 million[^2] with
\$679,409.19[^3] allocated for building the system.

They released [a web app](https://ghanapostgps.com/map/)
you can use to try out the system. It provides addresses for any chosen
location on the map.

![](https://i.imgur.com/BgWuDX5.png)
*The GhanaPostGPS web app, showing a location marker and address information on the right sidebar.*

In this article, I'll take you through my process of recreating the
end-to-end GhanaPostGPS system: data processing/generation, algorithms
and a bit of software engineering. Let's begin!

How does GhanaPostGPS work?
---------------------------

### The Address

Well, actually, before we begin, let's see how the GhanaPostGPS works.
According to the tabloids, the system gives unique addresses to every
5x5 m area in the country. An example of an address is **AK-039-5028**.

What does this address mean? First of all, the address is made up of
three parts.

![](https://i.imgur.com/2c1aGw0.png)
*Parts of the GhanaPost address: Regions \> Districts \> Areas \> Unique Locations*

The first part contains two letters representing the region and district
of that location, respectively. In this case, **A** for the **A**shanti
Region, and **K** for the **K**umasi District.

The second part, the area code, is a number representing a 500x500m zone
called 'area'. Each district can be divided into many areas. In this
case, the location is in an area with code **200**.

The third part, the unique address, is a number representing the very
5x5m zone containing a location. Each location (5x5m zone) is given a
unique number to ensure that all addresses are distinct. In this case,
the location is an area with the code **1987**.

These three parts come together to form what is called the GhanaPost
address. How many unique addresses can we allocate to the whole of
Ghana? 10 regions \* \~16 districts per region \* 10 <sup>3</sup> \* 10<sup>4</sup> = 1.6
billion addresses, cool!!

Now that we know what the address format is, let's look at how exactly
GhanaPostGPS might generate those codes.

### **The Algorithm** 

*Note*: The findings here are reflective of the research I have done
(gathered from articles, press releases, and inductive observation from
using the GhanaPost app itself), and *not* from the creators of the
GhanaPostGPS system. Therefore, this may not *fully* reflect how the
system works.

Let's look at how addresses are generated for each part separately.

#### **Part 1: Region and District code**

This part is relatively straightforward; each region and district is
given a distinct code (a letter) to represent it.

#### **Part 2: Area code**

Ugh, now it gets interesting! How does GhanaPost address every 500x500m
area in a district? According to
[this](https://www.myjoyonline.com/news/mahama-has-no-clue-about-digital-address-system-bawumia-comes-for-his-favourite-target/)
and
[this](https://www.ghanabusinessnews.com/2017/11/06/bawumia-replies-mahama-for-describing-ghana-post-gps-as-419-scam/),
the GhanaPost system uses a "Spiral Matrix Postcode algorithm" [^4]
which refers to the ['Spiral Matrix
algorithm](https://www.educative.io/edpresso/spiral-matrix-algorithm)'.
To label (or encode) all areas in a district, the algorithm starts at
the centre area (the area containing the approx. centre of the district,
with code **0**). It traverses around this centre in a spiralling
pattern labelling each cell it encounters. It does this until the whole
district is covered.

![](https://i.imgur.com/FA6PnrO.png)*Given a district, start at the center cell and move in a spiral pattern, numbering cells along the way*

*I believe the area code uses three digits because districts are
relatively small and can be covered in less than 1000 (10<sup>3</sup>) areas.*

#### Part 3: Unique address / location 

What about the unique address for each 5x5m location? How does GhanaPost
create codes for them? My response\...not sure. There is no public
information on how the team created them. My educated guesses range from
*randomly* assigning codes over every 5x5m location to using the *spiral
matrix algorithm* (mentioned in part 2) again but on areas this time.

Whew, that sums up how GhanaPost works *i.e* how it generates unique
addresses for every location in the country. In the next section, we'll
dive right into how I implemented the system and the decisions made
along the way to build such a system.

Implementation
---------------------------

### Step 0: Data Preparation

The first step to creating a system like GhanaPostGPS is to obtain
geometric information about the country, its regions and districts.
Because this project was experimental, I chose not to work with the
country's administrative regions and districts, but instead created
fictional ones with the help of my little siblings. We pretended to be
naive[^5] European leaders in the 19th century[^6] and divided the
country anyhow *we* pleased.

The geometric information (shapefile) for the country was downloaded
from the [Humanitarian Data Exchange](https://data.humdata.org/dataset/ghana-administrative-boundaries)
and manipulated using the open-source software,
[QGIS](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&ved=2ahUKEwjd5Pb809LqAhWUOcAKHetjCAYQFjAAegQIARAC&url=https%3A%2F%2Fqgis.org%2F&usg=AOvVaw2q1BgECmWBjBRYrizC7Phe).
I used QGIS to split up Ghana's polygon into four (4) region polygons
and further into ninety-four (94) district polygons. After which, we (my
little siblings & I) named each region and district using the power of
our imagination -you might see weird names as a result because there was
no editing process-

![](https://i.imgur.com/s94td9Z.png)*\[Ghana polygon
feature in QGIS\] → \[Divided into 4 regions polygons\] → \[Also
divided into 94 district polygons\]*

After the ~~fabrication~~ creation process, I wrote a [python
script](https://gist.github.com/kevin-deyoungster/11d3412dc86939d63290b6ab6e20a0f6)
to export all the features (district & regions) and their geometric
information (polygon coordinates, centre points) to individual GeoJSON
files.

![](https://i.imgur.com/QDjVCcs.png)*Screenshot of a few district GeoJSONs \| Screenshot of all region GeoJSONs*

Now that we have created the geometries for the country's regions and
districts, we proceed with generating addresses for all locations in
them.

### Step 1: Address Generation

#### Part 1: Region and District Code 

The codes for regions and districts should ideally be pre-determined and
stored in some mapping data structure like Python's dictionary where
names will be keys, and their respective codes will be the values. But
for this project, I went along with using the first letter of regions
and districts names to represent them (therefore no need for storage).

<table>
   <thead>
      <tr>
         <th>Region</th>
         <th>District</th>
         <th>Region and District Code (combined)</th>
      </tr>
   </thead>
   <tbody>
      <tr>
         <td><strong>L</strong>owaki</td>
         <td><strong>E</strong>dan</td>
         <td><strong>LE</strong></td>
      </tr>
      <tr>
         <td><strong>S</strong>outhowa</td>
         <td><strong>D</strong>abumbum</td>
         <td><strong>SD</strong></td>
      </tr>
      <tr>
         <td><strong>N</strong>orthupper</td>
         <td><strong>D</strong>ulolo</td>
         <td><strong>ND</strong></td>
      </tr>
      <tr>
         <td><strong>U</strong>peaster</td>
         <td><strong>T</strong>ipo</td>
         <td><strong>UT</strong></td>
      </tr>
      <tr>
         <td colspan="3">Table showing four sample districts from each region and their codes</td>
      </tr>
   </tbody>
</table>

#### Part 2: Area Code 

Earlier, I explained that GhanaPost used the Spiral Matrix algorithm to
label areas in a district. It's well suited for quadrilateral cells
(GhanaPost uses a square grid). In this recreation, I used a similarly
hierarchical (multiple-level) grid system but with hexagons instead of
squares. I used [Uber's H3](https://eng.uber.com/h3/)
spatial grid index because it was easily accessible (compared to [the
quadrilateral alternative](https://s2geometry.io)). Since
hexagon grids cells have six edge neighbours instead of four, I had to
use a different approach to spirally traversing areas in a region.

I used a Breadth-First-Search approach that starts at the centre cell of
the district and 'spirals' outward until the entire district polygon was
filled with area cells. The simplified pseudocode is as follows:

```python
# Initialization
queue = Queue()
code = 0
Find the center of the district, C (x,y)
Get the grid cell G0 that contains this center point
Add G0 to queue
While queue is not empty:
    Gn = pop the next item from the queue
    Label Gn with code
    code += 1
    Get all the neighbors of grid cell Gn
    Add only the valid neighbors to queue
```

The procedure is illustrated below:

![](https://i.imgur.com/er95xdL.png)*1.Find the center point of the district → 2. Label the center cell →1. Label the center cells immediate neighbours → 4. Label those neighbours' neighbours → 5. Keep going until the whole region is covered with grid cells*

#### Part 3: Location Code / Unique address 

Earlier, I explained that areas (500x500m zones) contain much smaller
locations (5x5m zones). Since areas and locations are at different
resolutions (sizes), a hierarchical grid system in relating the smaller
grid cells to their larger parent cells.

![](https://i.imgur.com/0rMDP9q.png)*H3 grid cells, at resolution for areas \| H3 grid cells, at resolution for locations*

To generate codes for locations, I used the same BFS spiralling
algorithm for each area but at a smaller resolution (grid cell size) to
correctly represent locations.

![](https://i.imgur.com/EsnC2kW.png)*7 areas (red hexagons) \| 300 locations (blue hexagons)*

#### Part 4: Storage

The labels/codes for the areas and location cells in each district were
stored in a simple SQLite database using the following schema (like a
nested JSON):

```javascript
{
    [district_name]: {
        [area_cell_address]: {
            id: int,
            children: {
                [unique_cell_address]: {
                    id: int
                }  
                ...
            }
        }
    ...
}
```

Because this project used larger regions (four massive ones) and
fairly-sized districts (ninety-four), I had to switch up the format of
the address to accommodate the larger number of areas per district (in
the thousands). So, for area codes, I used four digits to represent
rather than three, and for location codes (unique address) I used three
digits instead of four. The next address format was:
**XX**-**XXXX**-**XXX**

At this point, every location in the country has been correctly
addressed. Next was finding out how to fetch these addresses given a
user's location.

### Step 2: Coordinate Mapping 

Great, at this point we have a \~700MB datastore containing area codes
and unique location addresses for every district, waiting to be used.
Say someone at Accra Mall, waiting for their date to show up, wants to
check this system for their address, how would we know what the address
is?

Primarily, we're looking for a function *f* that takes in GPS
coordinates, *x* (latitude), *y* (longitude) and outputs an address *A*.

![](https://i.imgur.com/qRR8vgD.png)

The process of fetching the address for any given GPS coordinate is as
follows:

1.  Find which region and district it falls in and fetch their codes (first letters)
2.  Find which area *and* location it falls in and fetch their codes from the datastore
3.  Return the address by combining outputs from (1), (2)

#### 1. Find the region and district that contain the GPS coordinate

How do we find which of the 94 districts contains the input GPS
coordinate?

**Option A**: Go through every district (polygon) one by one and see if
the GPS coordinate 'fits in'

**Option B**: Use a tree-like index structure to check only a certain
number of possible regions to see if the coordinate 'fits in'

Which option would you pick?\...*\*click\**\...fantastico! Option B,
because it's much faster. Even though both options are correct, option
two is more efficient. Option A, in the worst-case scenario, will check
all *n* districts (94), but Option B will check max 29 districts!

The tree-like structure used in this project was the
[RTree](https://en.wikipedia.org/wiki/R-tree), a data
structure for very efficient spatial queries. Fortunately, the library
used to read the GeoJSON datasets for districts and regions (GeoPandas)
has an in-built in-memory implementation to perform Option B.

![](https://i.imgur.com/lRZs6G4.png)*With input coordinate (yellow circle), we use a spatial index (r-tree based) to obtain the region (black border) and district (red border)*

In this example, the person (yellow spot) was in the Numumoosh district
(red border) of the Lowaki region (black border) resulting in the codes
**LN**.

#### 2. Find the area code and location code (unique address)

Area codes and location codes (unique addresses) were earlier stored in
an SQLite database in a nested JSON structure, and tagged by their grid
cell addresses. So, to find the corresponding area and location codes,
we ask the grid system for the grid cell containing the input GPS
coordinate at the area resolution and location resolution. We then query
the database for the codes associated with these grid cells.

![](https://i.imgur.com/mcEHiE8.png)*Find the code for the area cell (blue hexagon) and location cell (green hexagon)*

In this example, the database coughed up **7915** for the area code, and
**002** for the location code/unique address.

#### 3. Put them all together! 

Now combine each component from step 1 (**LN**) and 2 (**7915**,
**002**) and voila! The final address!

![](https://i.imgur.com/OZ2aT9G.png)

The final address for this person in Accra Mall is **LN-7915-002**.
Hopefully, their date shows up.

With core functionality complete, we move on to building the application
users would interact with, in this next section.

### Step 3: Application Development

To demonstrate the solution, I built a replica of [the GhanaPost web
app](https://ghanapostgps.com/map/). Since the language
used so far was Python, I decided to make a minimal backend server
'around' it. I chose the minimal Flask framework because there were only
three endpoints needed for the app (one to serve the frontend, one to
return unique addresses for GPS coordinates and the last to generate
search suggestions using [geopy's
geocoding](https://towardsdatascience.com/reverse-geocoding-in-python-a915acf29eb6)
abilities).

The front end was built with React.js library because why not? (I needed
an opportunity to brush up on that skill) and Leaflet library was used
to render the maps from OpenStreetMaps.

Screenshots:
![](https://i.imgur.com/1ntpZyF.png)

![](https://i.imgur.com/ondhtht.png)

![](https://i.imgur.com/lRIB83o.png)

Conclusion 
-----------

![](https://i.imgur.com/c4a2Mu2.png)

There you have it, that's how I 'recreated' GhanaPostGPS, Ghana's
National Digital Address System. Check out the [live
demo](https://cutt.ly/ghanapostclone);
I hope this was insightful for you. If you have any comments or
questions, feel free to hit me up on my twitter
[@kev\_deyoungster](https://twitter.com/kev_deyoungster) 🤗. Also, if
you are from the team that built this system or work in the geospatial
industry, I'd be very excited to hear from you!

Acknowledgment
--------------

I give shout-outs to my friends Oracking, Jean-Sebastien and Boham for
taking the time to review this write-up, and my little siblings for
naming the regions and districts 🤗. Thanks to all the articles that
helped me understand bits and pieces of GhanaPost, especially
[Adiamah's](https://medium.com/@deladiamah/ghana-digital-addressing-system-revolutionary-or-ill-considered-acdd3360ff72). Final
appreciation to all the software engineers who work together to build
and maintain open-source projects without which this project would not
be possible, thank you all.

Disclaimer
----------

I do not work in the geospatial industry and therefore do not have the
expert knowledge on geospatial analysis. This recreation was made from
research and deductive reasoning. This is not a trivialization of the
actual system which demands meticulous attention to detail (and, of
course, has its [weaknesses](https://medium.com/@deladiamah/ghana-digital-addressing-system-revolutionary-or-ill-considered-acdd3360ff72)).
Lastly, this is a prototype, and its applications (backend and frontend)
were not made for scale, please abeg.

[^1]: [http://citifmonline.com/2017/10/nana-addo-launches-ghanas-digital-property-address-system/](http://citifmonline.com/2017/10/nana-addo-launches-ghanas-digital-property-address-system/)
[^2]: [https://www.pulse.com.gh/ece-frontpage/digital-addressing-system-ghana-post-blows-ghc35-million-on-publicity-to-market/n4bhrdm](https://www.pulse.com.gh/ece-frontpage/digital-addressing-system-ghana-post-blows-ghc35-million-on-publicity-to-market/n4bhrdm)
[^3]: [https://twitter.com/statsgh/status/1050062769456852993?s=20](https://twitter.com/statsgh/status/1050062769456852993?s=20)
[^4]: [https://www.myjoyonline.com/news/mahama-has-no-clue-about-digital-address-system-bawumia-comes-for-his-favourite-target/](https://www.myjoyonline.com/news/mahama-has-no-clue-about-digital-address-system-bawumia-comes-for-his-favourite-target/)
[^5]: [https://www.youtube.com/watch?v=sTa5iDbZXu0](https://www.youtube.com/watch?v=sTa5iDbZXu0)
[^6]: [https://www.ghanaweb.com/GhanaHomePage/features/A-short-history-of-the-creation-of-regions-in-Ghana-717336](https://www.ghanaweb.com/GhanaHomePage/features/A-short-history-of-the-creation-of-regions-in-Ghana-717336)
