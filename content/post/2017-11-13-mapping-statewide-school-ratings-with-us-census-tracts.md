---
title: Mapping Statewide School Ratings with US Census Tracts
author: Daniel Anderson
date: '2017-11-13'
slug: mapping-statewide-school-ratings-with-us-census-tracts
categories: [R Code]
tags:
  - Data Visualization
  - Mapping
  - Census Data
output: 
  html_document:
    keep_md: true
---


In this post, I'd like to share some work related to geo-spatial mapping, 
statewide school ratings, and US Census Bureau data using census tracts. 
Specifically, I wanted to investigate whether there was a relation between the
median housing price in an area, and the statewide achievement ratings for 
schools in the corresponding area. There is a 
[strong relation](https://steinhardt.nyu.edu/scmsAdmin/media/users/lec321/Sirin_Articles/Sirin_2005.pdf)
between socio-economic status and student achievement, but less is known about
how statewide ratings for schools relate to the demographics of the 
corresponding area. Fair warning - this is a rather long post, so you'll have to have a bit
of endurance to get through it all (I considered splitting it into two posts
but it felt less cohesive). We'll be using R to produce the following map.

<iframe seamless src="../2017-11-13-mapping-statewide-school-ratings-with-us-census-tracts_files/map_schools.html" width="100%" height="500"></iframe>

This post uses the [tidyverse](https://www.tidyverse.org), [Thomas 
Leeper](http://thomasleeper.com)'s [rio](https://github.com/leeper/rio) package 
for data import and [Sam Firke](https://github.com/sfirke)'s 
[janitor](https://github.com/sfirke/janitor) package for quick cleaning up of
column names. The [leaflet](https://rstudio.github.io/leaflet/) packages, is 
used for mapping, as well the terrific 
[tidycensus](https://github.com/walkerke/tidycensus) package by 
[Kyle Walker](https://walkerke.github.io), and the  
[ggmap](https://github.com/dkahle/ggmap) package is used for pulling 
lattitude and longitude data from physical addresses. Finally, this post 
builds off a [wonderful post](https://juliasilge.com/blog/using-tidycensus/) by 
[Julia Silge](https://juliasilge.com).

## The data
For this project, I wanted to use only publicly available data. I meandered 
around the Oregon Department of Education (ODE) website for a while and eventually
came to the file I wanted: [Report Card Ratings for each school](http://www.oregon.gov/ode/schools-and-districts/reportcards/reportcards/Pages/Accountability-Measures.aspx).
So let's import the data directly from the ODE website, clean it up a bit, and 
take a look.


```r
# load packages for data import/prep
library(tidyverse)
library(rio)
library(janitor)

ratings <- import("http://www.oregon.gov/ode/schools-and-districts/reportcards/reportcards/Documents/rcdetails_1617.xlsx",
                  setclass = "tbl_df") %>% 
            clean_names()

# Clean up the file to what we need
achievement <- ratings %>% 
  filter(district_name == "Portland SD 1J") %>% 
  spread(indicator, rating) %>% 
  clean_names() %>% 
  select(school_name, achievement) %>% 
  mutate(achievement = parse_number(achievement)) %>% 
  filter(!is.na(achievement))
	
achievement
```

```r
## # A tibble: 84 x 2
##    school_name                   achievement
##    <chr>                               <dbl>
##  1 Abernethy Elementary School          5.00
##  2 Ainsworth Elementary School          5.00
##  3 Alameda Elementary School            5.00
##  4 Arleta Elementary School             3.00
##  5 Astor Elementary School              3.00
##  6 Atkinson Elementary School           4.00
##  7 Rosa Parks Elementary School         2.00
##  8 Beach Elementary School              3.00
##  9 Beaumont Middle School               4.00
## 10 Boise-Eliot Elementary School        2.00
## # ... with 74 more rows
```

In the above code, we import the ratings, filter for only schools in Portland
School District, spread the data out so we have different columns for each type
of rating (achievement, growth, and student subgroup growth), and then select 
only the two variables we really need - the school name and the achievement 
rating.

This is a good start, but we don't yet have the geographical coordinates of the 
schools. To do that, we'll get another dataset from ODE that has the physical 
address of each school. We'll then transform those addresses to lattitude and 
longitude using a bit of help from the *ggmap* package 
First, the addresses:


```r
addresses <- import("http://www.ode.state.or.us/pubs/labels/SchoolMail.xls",
                    setclass = "tbl_df") %>% 
  clean_names() %>% 
  filter(city == "PORTLAND") %>% 
  mutate(name = str_to_title(name)) %>% 
  unite(address, c(address, city, state, zip), sep = " ") %>% 
  select(name, address)
              
addresses
```

```r
## # A tibble: 145 x 2
##    name                        address                                    
##    <chr>                       <chr>                                      
##  1 Abernethy Elementary School 2421 SE ORANGE AVE PORTLAND OR 97214-5398  
##  2 Ainsworth Elementary School 2425 SW VISTA AVE PORTLAND OR 97201-2399   
##  3 Alameda Elementary School   2732 NE FREMONT ST PORTLAND OR 97212-2542  
##  4 Alder Elementary School     17200 SE ALDER ST PORTLAND OR 97233-4260   
##  5 Alice Ott Middle School     12500 SE RAMONA ST PORTLAND OR 97236-4699  
##  6 Alliance High School        4039 NE ALBERTA CT PORTLAND OR 97211-8199  
##  7 Arleta Elementary School    5109 SE 66TH AVE PORTLAND OR 97206-4599    
##  8 Arthur Academy              13717 SE DIVISION ST PORTLAND OR 97236-2841
##  9 Astor Elementary School     5601 N YALE ST PORTLAND OR 97203-5299      
## 10 Atkinson Elementary School  5800 SE DIVISION ST PORTLAND OR 97206-1499 
## # ... with 135 more rows
```
After import, we limit to only schools in Portland because we're going to be 
joining this dataset with our *ratings* dataset based on the school name, and
we want to ensure we don't join the data for schools with the same name but 
but different districts. Unfortunately, the *addresses* file doesn't tell us the
district, or we could include that as one of the keyed variables in our join.

We'll now join the *addresses* dataset with our *ratings* dataset.


```r
d <- inner_join(achievement, addresses, by = c("school_name" = "name"))
```

Now, we can find the longitude and lattitude of each school using the physical
address with the help of the *ggmap* package and, specifically, the `geocode`
function. This takes a bit of time. Notice I've changed the source from the default, which is google, to "dsk", which uses [data science toolkit](http://www.datasciencetoolkit.org) instead. I like this better because it tends to provide better results, at least in my experience, unless their servers are being overwhelmed (which is not terrifically infrequent), in which case it fails catastrophically.


```r
library(ggmap)
lat_long <- geocode(d$address, source = "dsk")
d <- bind_cols(d, lat_long)
d[ ,c(1, 4:5)]
```

```r
## # A tibble: 81 x 3
##    school_name                     lon   lat
##    <chr>                         <dbl> <dbl>
##  1 Abernethy Elementary School    -123  45.5
##  2 Ainsworth Elementary School    -123  45.5
##  3 Alameda Elementary School      -123  45.5
##  4 Arleta Elementary School       -123  45.5
##  5 Astor Elementary School        -123  45.6
##  6 Atkinson Elementary School     -123  45.5
##  7 Rosa Parks Elementary School   -123  45.6
##  8 Beach Elementary School        -123  45.6
##  9 Beaumont Middle School         -123  45.5
## 10 Boise-Eliot Elementary School  -123  45.5
## # ... with 71 more rows
```

Note - an earlier draft of this post had specified `source = "dsk"` when running
the `geocode` function. I was getting some missing values (~19%) when using 
google (the default) and I received no missing data with `source = "dsk"`. 
However, it seems the Data Science Toolkit website is fairly frequently 
overwhelmed. Last time I tried this it took forever and failed, so I suppose
the best option may depend on the availability of the servers at time you run 
the function.

## Mapping the data
To map the data, we'll use the *leaflet* and *sf* packages. We'll first write a 
quick function that tells leaflet what color we want the schools in our map to 
appear depending on their statewide achievement rating. We'll make a simple 
diverging palette, where low values are red, middle values are white, and high 
values are green. These colors must correspond with the possible values for
`markerColor` from `leaflet::awesomeIcons`.


```r
library(leaflet)

get_col <- function(df, indicator) {
  sapply(df[[as.character(indicator)]], function(rating) {
    if(rating == 5) {
      "green"
    } else if(rating == 4) {
      "lightgreen"
    } else if(rating == 3) {
      "white"
    } else if(rating == 2) {
      "lightred"
    } else {
      "red"
    } })
}
```


Next we'll map the Portland area using a combination of *leaflet* and the 
*tidycensus* packages to not only produce the map, but color census tracts
according to the median housing income for the tract. Note that you have to have 
a US Census API key for these function to work (see 
[here](https://walkerke.github.io/tidycensus/articles/basic-usage.html)).
Below, I use the  `get_acs` function from *tidycensus* package to pull census 
tract data for  Multnomah County. The variable argument tells the function 
which variable to get data from (see the `load_variables` function for other
variables). I also filter the data for non-negative values (there was one that 
was returning a negative value, I haven't taken the time to track down why yet).


```r
library(tidycensus)
pdx_acs <- get_acs(geography = "tract", 
                     variables = "B25077_001", 
                     state = "OR",
                     county = "Multnomah County",
                     geometry = TRUE) %>% 
  filter(estimate > 0) 
```

The `pdx_acs` object now has data on the median housing cost for all census 
tracts in Multnomah County, as well as the geometry of each census tract. We
can use this information to produce the map. First, however, we need to create
a color palette for the tracts. We'll use the viridis palette, which is not 
only beautiful, but also good for people with color blindness, as well as for
printing in black and white. We'll use the `colorNumeric` function from 
*leaflet*, and ask it to fill each tract according to the estimate of the 
median housing cost, which we just extracted with the *tidycensus* package.


```r
pal <- colorNumeric(palette = "viridis", 
                    domain = pdx_acs$estimate)
```

Next we'll get the actual icons we want to use, coloring them according to
the achievement ratings. The last line in this code gets a hexadecimal 
approximation for these colors, which we'll use in a legend later.


```r
pins <- awesomeIcons(
  icon = 'ios-close',
  iconColor = 'black',
  library = 'ion',
  markerColor = get_col(d, "achievement")
)

cols <- gplots::col2hex(c("red", "pink", "white", "lightgreen", "darkgreen"))
```

Finally, we'll produce the map, using the code below. Although it appears
somewhat complicated, it's actually not too bad, it's just rather long(ish). 
Essentially we first separate `NAME` into three distinct variables for the
census tract, county, and state, then we (a) create a blank map, (b) set the 
style in which we want the map to render, (c) add tract information, 
(d) add school information, and (e) add legends.


```r
pdx_acs %>%
  separate(NAME, c("tract", "county", "state"), sep = ",") %>% 
  leaflet() %>%
  addProviderTiles(provider = "Stamen.TerrainBackground") %>%
  addPolygons(popup = ~ tract,
              stroke = FALSE,
              smoothFactor = 0,
              fillOpacity = 0.7,
              color = ~ pal(estimate)) %>%
  addAwesomeMarkers(data = d, 
                    ~lon, 
                    ~lat, 
                    icon = pins, 
                    popup= ~school_name) %>% 
  addLegend("bottomright", 
            pal = pal, 
            values = ~ estimate,
            title = "Median Home Value",
            labFormat = labelFormat(prefix = "$"),
            opacity = 1) %>% 
  addLegend("bottomright", 
            colors = rev(cols),
            labels = 5:1,
            title = "Achievement Ratings",
            opacity = 1)
```





<iframe seamless src="../2017-11-13-mapping-statewide-school-ratings-with-us-census-tracts_files/map_schools.html" width="100%" height="500"></iframe>

One of the nice things about this map is that it's fully interactive. And
that's particularly helpful here because to really see the relation you'll need
to zoom in quite a bit. Because we added the optional `popup` argument to 
`addPolygons` and `addAwesomeMarkers`, we're able to click to identify a 
specific census tract or the name of a specific school. If we wanted the plot
to render differently we could change the provider. For example, you could 
produce a nice terrain map with `provider = "Stamen.TerrainBackground"`. All
the different options are available 
[here](http://leaflet-extras.github.io/leaflet-providers/preview/index.html), 
where you can preview a map before applying it to yours.

Note that we have a very clear and very strong relation here - essentially all
the red, pink, and white schools are in the more purple tracts, correpsonding to
lower median housing price areas. This is powerful information but it is 
important that it's not confused to mean that the lowest quality schools are 
located in the more impoverished areas. This information tells us very little
about the quality of the school and, indeed, if we look at estimates of growth
(shown below without all the code) there is a much less strong correlation. That
is, the ratings schools receive on the amount of growth students make is less
correlated with the median housing cost in the area than is absolute achievement,
although there does still appear to be some correlation.



<iframe seamless src="../2017-11-13-mapping-statewide-school-ratings-with-us-census-tracts_files/map_schools_growth.html" width="100%" height="500"></iframe>
