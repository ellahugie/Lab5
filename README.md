Lab 05 - Data Wrangling
================
Ella Hugie

# Learning goals

- Use the `merge()` function to join two datasets.
- Deal with missings and impute data.
- Identify relevant observations using `quantile()`.
- Practice your GitHub skills.

# Lab description

For this lab we will be dealing with the meteorological dataset `met`.
In this case, we will use `data.table` to answer some questions
regarding the `met` dataset, while at the same time practice your
Git+GitHub skills for this project.

This markdown document should be rendered using `github_document`
document.

# Part 1: Setup a Git project and the GitHub repository

1.  Go to wherever you are planning to store the data on your computer,
    and create a folder for this project

2.  In that folder, save [this
    template](https://github.com/JSC370/JSC370-2024/blob/main/labs/lab05/lab05-wrangling-gam.Rmd)
    as “README.Rmd”. This will be the markdown file where all the magic
    will happen.

3.  Go to your GitHub account and create a new repository of the same
    name that your local folder has, e.g., “JSC370-labs”.

4.  Initialize the Git project, add the “README.Rmd” file, and make your
    first commit.

5.  Add the repo you just created on GitHub.com to the list of remotes,
    and push your commit to origin while setting the upstream.

Most of the steps can be done using command line:

``` sh
# Step 1
cd ~/Documents
mkdir JSC370-labs
cd JSC370-labs

# Step 2
wget https://raw.githubusercontent.com/JSC370/JSC370-2024/main/labs/lab05/lab05-wrangling-gam.Rmd
mv lab05-wrangling-gam.Rmd README.Rmd
# if wget is not available,
curl https://raw.githubusercontent.com/JSC370/JSC370-2024/main/labs/lab05/lab05-wrangling-gam.Rmd --output README.Rmd

# Step 3
# Happens on github

# Step 4
git init
git add README.Rmd
git commit -m "First commit"

# Step 5
git remote add origin git@github.com:[username]/JSC370-labs
git push -u origin master
```

You can also complete the steps in R (replace with your paths/username
when needed)

``` r
# Step 1
setwd("~/Documents")
dir.create("JSC370-labs")
setwd("JSC370-labs")

# Step 2
download.file(
  "https://raw.githubusercontent.com/JSC370/JSC370-2024/main/labs/lab05/lab05-wrangling-gam.Rmd",
  destfile = "README.Rmd"
  )

# Step 3: Happens on Github

# Step 4
system("git init && git add README.Rmd")
system('git commit -m "First commit"')

# Step 5
system("git remote add origin git@github.com:[username]/JSC370-labs")
system("git push -u origin master")
```

Once you are done setting up the project, you can now start working with
the MET data.

## Setup in R

1.  Load the `data.table` (and the `dtplyr` and `dplyr` packages if you
    plan to work with those).

``` r
library("data.table")
library("leaflet")
fn <- "https://raw.githubusercontent.com/JSC370/JSC370-2024/main/data/met_all_2023.gz"
if (!file.exists("E:/UofT/Year 3/JSC370/Lab5/met_all_2023.gz"))
  download.file(fn, destfile = "E:/UofT/Year 3/JSC370/Lab5/met_all_2023.gz")
met <- data.table::fread("met_all_2023.gz")
```

2.  Load the met data from
    <https://github.com/JSC370/JSC370-2024/main/data/met_all_2023.gz> or
    (Use
    <https://raw.githubusercontent.com/JSC370/JSC370-2024/main/data/met_all_2023.gz>
    to download programmatically), and also the station data. For the
    latter, you can use the code we used during lecture to pre-process
    the stations data:

``` r
download.file(
  "https://raw.githubusercontent.com/JSC370/JSC370-2024/main/data/met_all_2023.gz",
  destfile = "met_all_2023.gz",
  method   = "curl",
  timeout  = 60
  )

met <- data.table::fread("met_all_2023.gz")
met$lat <- met$lat / 1000
met$lon <- met$lon / 1000
met$temp <- met$temp / 10
```

``` r
# Download the data
stations <- fread("ftp://ftp.ncdc.noaa.gov/pub/data/noaa/isd-history.csv")
stations[, USAF := as.integer(USAF)]

# Dealing with NAs and 999999
stations[, USAF   := fifelse(USAF == 999999, NA_integer_, USAF)]
stations[, CTRY   := fifelse(CTRY == "", NA_character_, CTRY)]
stations[, STATE  := fifelse(STATE == "", NA_character_, STATE)]

# Selecting the three relevant columns, and keeping unique records
stations <- unique(stations[, list(USAF, CTRY, STATE)])

# Dropping NAs
stations <- stations[!is.na(USAF)]

# Removing duplicates
stations[, n := 1:.N, by = .(USAF)]
stations <- stations[n == 1,][, n := NULL]
```

3.  Merge the data as we did during the lecture.

``` r
# merging data
met <- merge(
  # Data
  x     = met,      
  y     = stations, 
  # List of variables to match
  by.x  = "USAFID",
  by.y  = "USAF", 
  # Which obs to keep?
  all.x = TRUE,      
  all.y = FALSE
  )

# check top rows
head(met[, list(USAFID, WBAN, STATE)], n = 4)
```

    ##    USAFID  WBAN STATE
    ## 1: 690150 93121    CA
    ## 2: 690150 93121    CA
    ## 3: 690150 93121    CA
    ## 4: 690150 93121    CA

## Question 1: Representative station for the US

Across all weather stations, what is the median station in terms of
temperature, wind speed, and atmospheric pressure? Look for the three
weather stations that best represent continental US using the
`quantile()` function. Do these three coincide?

    ## Median Quantile Temperature: 21.7

    ## Median Quantile Wind Speed: 31

    ## Median Quantile Atmospheric Pressure: 10117

    ## Station representing the median Quantile Temperature: 690150

    ## Station representing the median Quantile Wind Speed: 690150

    ## Station representing the median Quantile Atmospheric Pressure: 690150

Yes, these three variables’ median values do all coincide at the same
station 690150. This is particularly interesting because it indicates a
possible relationship between these three variables temperature, wind
speed, and atmospheric pressure.

Knit the document, commit your changes, and save it on GitHub. Don’t
forget to add `README.md` to the tree, the first time you render it.

## Question 2: Representative station per state

Just like the previous question, you are asked to identify what is the
most representative, the median, station per state. This time, instead
of looking at one variable at a time, look at the euclidean distance. If
multiple stations show in the median, select the one located at the
lowest latitude.

``` r
# calculate Euclidean distance by state
met[, euclidean_distance := sqrt((lat - mean(lat))^2 + (lon - mean(lon))^2), by = STATE]

# find the median station for each state
median_stations <- met[, .SD[which.min(abs(euclidean_distance - median(euclidean_distance)))], by = STATE]

# if a tie, select the one with the lowest latitude
most_representative_stations <- median_stations[, .SD[which.min(lat)], by = STATE]
most_representative_stations <- most_representative_stations[, .(USAFID, WBAN, STATE, lat, lon)]

head(most_representative_stations)
```

    ##    USAFID  WBAN STATE    lat      lon
    ## 1: 720267 23224    CA 38.955 -121.081
    ## 2: 722588 13926    TX 33.068  -96.065
    ## 3: 726375 94817    MI 42.663  -83.410
    ## 4: 720596   189    SC 34.300  -81.633
    ## 5: 722089 94959    IL 40.933  -90.433
    ## 6: 720306 53879    MO 38.958  -94.371

Knit the doc and save it on GitHub.

## Question 3: In the middle?

For each state, identify what is the station that is closest to the
mid-point of the state. Combining these with the stations you identified
in the previous question, use `leaflet()` to visualize all ~100 points
in the same figure, applying different colors for those identified in
this question.

``` r
# converting lat and lon columns to numeric
met$lat <- as.numeric(met$lat)
met$lon <- as.numeric(met$lon)

# removing na rows
met <- met[!is.na(met$lat) & !is.na(met$lon), ]
```

``` r
# calculating midpoint of each state
midpoints <- met[, .(mid_lat = mean(lat), mid_lon = mean(lon)), by = STATE]

euclidean_distance <- function(lat1, lon1, lat2, lon2) {
  sqrt((lat1 - lat2)^2 + (lon1 - lon2)^2)
}

# finding closest station to the midpoint
find_closest_station <- function(mid_lat, mid_lon, stations) {
  distances <- euclidean_distance(stations$lat, stations$lon, mid_lat, mid_lon)
  closest_index <- which.min(distances)
  closest_station <- stations[closest_index, ]
  return(closest_station)
}

closest_stations <- lapply(1:nrow(midpoints), function(i) {
  state_midpoint <- midpoints[i, ]
  state_code <- state_midpoint$STATE
  state_stations <- met[STATE == state_code]
  closest_station <- find_closest_station(state_midpoint$mid_lat, state_midpoint$mid_lon, state_stations)
  return(closest_station[, .(USAFID, WBAN, STATE, lat, lon)])  # Select only the columns you need
})

# combine midpoint stations with previously identified stations
all_stations <- rbind(most_representative_stations, do.call(rbind, closest_stations))

map <- leaflet() %>%
  addTiles() %>%
  addCircleMarkers(data = all_stations, 
                   lat = ~lat, 
                   lng = ~lon,
                   color = ifelse(all_stations$USAFID %in% most_representative_stations$USAFID, "blue", "red"),
                   radius = 5,
                   popup = ~paste("Station:", USAFID, "<br>State:", STATE))

map
```

<div class="leaflet html-widget html-fill-item" id="htmlwidget-b95782f0fc7bddf6e45f" style="width:672px;height:480px;"></div>
<script type="application/json" data-for="htmlwidget-b95782f0fc7bddf6e45f">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addTiles","args":["https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",null,null,{"minZoom":0,"maxZoom":18,"tileSize":256,"subdomains":"abc","errorTileUrl":"","tms":false,"noWrap":false,"zoomOffset":0,"zoomReverse":false,"opacity":1,"zIndex":1,"detectRetina":false,"attribution":"&copy; <a href=\"https://openstreetmap.org/copyright/\">OpenStreetMap<\/a>,  <a href=\"https://opendatacommons.org/licenses/odbl/\">ODbL<\/a>"}]},{"method":"addCircleMarkers","args":[[38.955,33.068,42.663,34.3,40.933,38.958,34.545,45.417,46.683,34.272,44.123,31.356,40.948,35.937,39.143,42.219,40.238,41.964,46.375,42.887,38.38,39.417,35.3,34.65,41.444,30.521,37.09,28.483,41.563,40.683,36.45,39.072,48.301,44.567,38.533,33.873,41.483,39.183,38.417,44.016,36.056,42.57,41.53,42.555,38.69,43.627,44.05,47.326,36.333,31.106,43.622,33.921,39.835,38.096,35.257,44.095,47.28,32.64,45.949,33.178,40.711,35.582,37.358,41.691,40.849,40.967,44.894,44.359,39,38.981,33.612,35.358,43.062,30.558,37.578,28.474,40.28,40.277,35.003,38.068,48.39,44.205,39.05,32.32,41.51,38.05,40.219,44.382,36.009,42.207,41.597,42.191,39.133,43.567,44.798,47.054],[-121.081,-96.065,-83.41,-81.633,-90.43300000000001,-94.371,-94.203,-123.817,-122.983,-83.83,-93.261,-85.751,-87.18300000000001,-77.547,-78.14400000000001,-92.026,-75.55500000000001,-100.568,-117.015,-90.236,-81.59099999999999,-77.383,-112.2,-98.40000000000001,-106.827,-90.41800000000001,-84.069,-80.56699999999999,-83.477,-74.169,-105.666,-95.626,-102.406,-72.017,-106.933,-88.48999999999999,-73.133,-119.733,-110.7,-97.086,-85.53100000000001,-77.714,-71.283,-71.75700000000001,-75.36199999999999,-72.30500000000001,-70.283,-106.948,-119.95,-98.196,-84.73699999999999,-80.801,-88.866,-92.553,-93.095,-121.2,-121.34,-83.592,-94.34699999999999,-86.782,-86.375,-79.101,-78.438,-93.566,-77.849,-98.31699999999999,-116.099,-89.837,-80.274,-76.922,-111.923,-96.943,-108.447,-92.099,-84.77,-82.45399999999999,-83.11499999999999,-74.816,-105.662,-97.861,-100.024,-72.565,-105.516,-90.078,-72.828,-117.09,-111.723,-100.286,-86.52,-75.98,-71.41200000000001,-71.173,-75.467,-71.43300000000001,-68.819,-109.457],5,null,null,{"interactive":true,"className":"","stroke":true,"color":["blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red"],"weight":5,"opacity":0.5,"fill":true,"fillColor":["blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","blue","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red","red"],"fillOpacity":0.2},null,null,["Station: 720267 <br>State: CA","Station: 722588 <br>State: TX","Station: 726375 <br>State: MI","Station: 720596 <br>State: SC","Station: 722089 <br>State: IL","Station: 720306 <br>State: MO","Station: 720172 <br>State: AR","Station: 720202 <br>State: OR","Station: 720254 <br>State: WA","Station: 722185 <br>State: GA","Station: 726568 <br>State: MN","Station: 722239 <br>State: AL","Station: 744660 <br>State: IN","Station: 720864 <br>State: NC","Station: 724053 <br>State: VA","Station: 720326 <br>State: IA","Station: 725109 <br>State: PA","Station: 722211 <br>State: NE","Station: 727830 <br>State: ID","Station: 726507 <br>State: WI","Station: 724140 <br>State: WV","Station: 722081 <br>State: MD","Station: 720635 <br>State: AZ","Station: 723550 <br>State: OK","Station: 720521 <br>State: WY","Station: 722312 <br>State: LA","Station: 724243 <br>State: KY","Station: 747940 <br>State: FL","Station: 724287 <br>State: OH","Station: 725020 <br>State: NJ","Station: 723663 <br>State: NM","Station: 724560 <br>State: KS","Station: 720909 <br>State: ND","Station: 720492 <br>State: VT","Station: 724677 <br>State: CO","Station: 746941 <br>State: MS","Station: 725029 <br>State: CT","Station: 720549 <br>State: NV","Station: 724733 <br>State: UT","Station: 720624 <br>State: SD","Station: 723274 <br>State: TN","Station: 724988 <br>State: NY","Station: 725079 <br>State: RI","Station: 725107 <br>State: MA","Station: 724093 <br>State: DE","Station: 726116 <br>State: NH","Station: 726184 <br>State: ME","Station: 727684 <br>State: MT","Station: 747020 <br>State: CA","Station: 720647 <br>State: TX","Station: 725424 <br>State: MI","Station: 723105 <br>State: SC","Station: 725316 <br>State: IL","Station: 724459 <br>State: MO","Station: 723429 <br>State: AR","Station: 720638 <br>State: OR","Station: 727815 <br>State: WA","Station: 722175 <br>State: GA","Station: 726578 <br>State: MN","Station: 722300 <br>State: AL","Station: 720961 <br>State: IN","Station: 722201 <br>State: NC","Station: 724017 <br>State: VA","Station: 725466 <br>State: IA","Station: 725128 <br>State: PA","Station: 725520 <br>State: NE","Station: 725864 <br>State: ID","Station: 726452 <br>State: WI","Station: 720328 <br>State: WV","Station: 722244 <br>State: MD","Station: 722789 <br>State: AZ","Station: 722187 <br>State: OK","Station: 726720 <br>State: WY","Station: 720468 <br>State: LA","Station: 720448 <br>State: KY","Station: 722014 <br>State: FL","Station: 720928 <br>State: OH","Station: 724095 <br>State: NJ","Station: 722677 <br>State: NM","Station: 724506 <br>State: KS","Station: 720867 <br>State: ND","Station: 726145 <br>State: VT","Station: 726396 <br>State: CO","Station: 722350 <br>State: MS","Station: 725027 <br>State: CT","Station: 724855 <br>State: NV","Station: 725724 <br>State: UT","Station: 726560 <br>State: SD","Station: 723273 <br>State: TN","Station: 725150 <br>State: NY","Station: 725074 <br>State: RI","Station: 725098 <br>State: MA","Station: 724088 <br>State: DE","Station: 726155 <br>State: NH","Station: 726070 <br>State: ME","Station: 726776 <br>State: MT"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null]}],"limits":{"lat":[28.474,48.39],"lng":[-123.817,-68.819]}},"evals":[],"jsHooks":[]}</script>

Knit the doc and save it on GitHub.

## Question 4: Means of means

Using the `quantile()` function, generate a summary table that shows the
number of states included, average temperature, wind-speed, and
atmospheric pressure by the variable “average temperature level,” which
you’ll need to create.

Start by computing the states’ average temperature. Use that measurement
to classify them according to the following criteria:

- low: temp \< 20
- Mid: temp \>= 20 and temp \< 25
- High: temp \>= 25

``` r
# find average temperature for each state
average_temp_by_state <- met[, .(avg_temp = mean(temp, na.rm = TRUE)), by = STATE]

# classify states
average_temp_by_state[, temp_level := cut(avg_temp,
                                         breaks = c(-Inf, 20, 25, Inf),
                                         labels = c("Low", "Mid", "High"),
                                         right = FALSE)]

# look at first few rows
head(average_temp_by_state)
```

    ##    STATE avg_temp temp_level
    ## 1:    CA 18.33406        Low
    ## 2:    TX 27.79978       High
    ## 3:    MI 18.53920        Low
    ## 4:    SC 23.25221        Mid
    ## 5:    IL 22.08465        Mid
    ## 6:    MO 23.85532        Mid

Once you are done with that, you can compute the following:

- Number of entries (records),
- Number of NA entries,
- Number of stations,
- Number of states included, and
- Mean temperature, wind-speed, and atmospheric pressure.

All by the levels described before.

``` r
# merge with met to recover information
met_merge <- merge(met, average_temp_by_state, by = "STATE")
setDT(met_merge)

met_merge$temp_level <- factor(met_merge$temp_level, levels = c("Low", "Mid", "High"))

# computer statistics
summary_table <- met_merge[, .(
  Number_of_Entries = .N,                                # Number of entries (records)
  Number_of_NA_Entries = sum(is.na(temp)),               # Number of NA entries
  Number_of_Stations = length(unique(USAFID)),           # Number of stations
  Number_of_States = length(unique(STATE)),              # Number of states
  Mean_Temperature = mean(temp, na.rm = TRUE),           # Mean temperature
  Mean_Wind_Speed = mean(wind.sp, na.rm = TRUE),         # Mean wind-speed
  Mean_Atmospheric_Pressure = mean(atm.press, na.rm = TRUE) # Mean atmospheric pressure
), by = temp_level]

summary_table
```

    ##    temp_level Number_of_Entries Number_of_NA_Entries Number_of_Stations
    ## 1:        Mid           1275529                29889                836
    ## 2:       High            465366                16221                342
    ## 3:        Low            828528                22715                674
    ##    Number_of_States Mean_Temperature Mean_Wind_Speed Mean_Atmospheric_Pressure
    ## 1:               22         22.21188        35.53520                  10114.74
    ## 2:                5         27.16140        39.15622                  10105.10
    ## 3:               21         17.69560        35.92808                  10123.70

Knit the document, commit your changes, and push them to GitHub.

## Question 5: Advanced Regression

Let’s practice running regression models with smooth functions on X. We
need the `mgcv` package and `gam()` function to do this.

- using your data with the median values per station, examine the
  association between median temperature (y) and median wind speed (x).
  Create a scatterplot of the two variables using ggplot2. Add both a
  linear regression line and a smooth line.

- fit both a linear model and a spline model (use `gam()` with a cubic
  regression spline on wind speed). Summarize and plot the results from
  the models and interpret which model is the best fit and why.

``` r
#install.packages("mgcv")
library(mgcv)
library(ggplot2)

met_median <- met[, .(
  median_temperature = median(temp, na.rm = TRUE),
  median_wind_speed = median(wind.sp, na.rm = TRUE)
), by = .(USAFID)]

# scatterplot of median temperature (y) vs. median wind speed (x)
ggplot(data = met_median, aes(x = median_wind_speed, y = median_temperature)) +
  geom_point() + 
  geom_smooth(method = "lm", se = FALSE, color = "blue") +  # Add linear regression line
  geom_smooth(method = "gam", formula = y ~ s(x, bs = "cs"), color = "red") +  # Add smooth line with cubic regression spline
  labs(x = "Median Wind Speed", y = "Median Temperature") +
  theme_minimal()
```

![](README_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

``` r
# linear model and a spline model using gam()
linear_model <- lm(median_temperature ~ median_wind_speed, data = met_median)
spline_model <- gam(median_temperature ~ s(median_wind_speed, bs = "cs"), data = met_median)

# summarize the results
summary(linear_model)
```

    ## 
    ## Call:
    ## lm(formula = median_temperature ~ median_wind_speed, data = met_median)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -23.5184  -2.7288   0.0712   2.7851  12.8996 
    ## 
    ## Coefficients:
    ##                   Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)       18.79692    0.46215  40.673  < 2e-16 ***
    ## median_wind_speed  0.07430    0.01376   5.399 7.57e-08 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 4.284 on 1837 degrees of freedom
    ##   (13 observations deleted due to missingness)
    ## Multiple R-squared:  0.01562,    Adjusted R-squared:  0.01508 
    ## F-statistic: 29.15 on 1 and 1837 DF,  p-value: 7.573e-08

``` r
summary(spline_model)
```

    ## 
    ## Family: gaussian 
    ## Link function: identity 
    ## 
    ## Formula:
    ## median_temperature ~ s(median_wind_speed, bs = "cs")
    ## 
    ## Parametric coefficients:
    ##             Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept) 21.23306    0.09905   214.4   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Approximate significance of smooth terms:
    ##                        edf Ref.df     F p-value    
    ## s(median_wind_speed) 6.985      9 7.229  <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## R-sq.(adj) =  0.0319   Deviance explained = 3.56%
    ## GCV = 18.121  Scale est. = 18.043    n = 1839

The linear model is: median_temperature = 18.79692 + 0.07430 \*
median_wind_speed

Note that the coefficient of median_wind_speed is statistically
significant meaning there is likely a relationship between
median_wind_speed and median_temperature. Similarly, the F-statistic is
very small indicating this model is overall statistically significant.
However, the R2 value of 0.01562 is extremely low meaning that only
about 1.562% of the variability in median temperature is explained by
median wind speed.

The spline model suggests a nonlinear relationship between median
temperature and median wind speed. This model has a slightly higher R2
value of 0.0319 indicating this model better explain the variability in
median temperature. Also, the degrees of freedom 6.985 indicated that
the spline is fairly flexible.

Overall, the spline appears to be a better fit due to its flexibility
and being able to explain more variation in median temperature, our
variable of interest. However, it should be noted that neither model
does a particularly good job and potentially another model should be
fitted to better explain the variation in median temperature.

## Deliverables

- .Rmd file (this file)

- link to the .md file (with all outputs) in your GitHub repository
