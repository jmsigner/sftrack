---
title: "4. Fantastic Bursts and how to use them"
output:
  pdf_document: default
  html_document: default
vignette: |
  %\VignetteIndexEntry{4. Fantastic Bursts and how to use them}
   %\VignetteEncoding{UTF-8}
   %\VignetteEngine{knitr::rmarkdown}
---
```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
devtools::load_all("/home/matt/r_programs/sftrack")
#library(sftrack)

```

## Bursts  

Bursts are a big emphasis in `sftrack`. They are made in a similar vein to the `sfc` and `sfg` in `sf`. 

To begin an `ind_burst` is a singular burst. Its whats stored at the row level. 
A `multi-burst` is a collection of `ind_bursts` and exists at the column level. Bursts also have an `active_burst` argument, which turns on and off certain bursts for analysis and plotting purposes.

We start by looking at the structure of a `multi_burst`
```{r}
data <- read.csv(system.file('extdata/raccoon_data.csv', package='sftrack'))
burst_list = list(id = data$sensor_code, month = as.POSIXlt(data$acquisition_time)$mon+1)

mb1 <- make_multi_burst(x=burst_list, active_burst=c('id','month'))
str(mb1)
mb1[[1]]
```

A burst contains grouping information. The `id` of the subject/sensor is the lowest level of grouping for the data. Any additional grouping variables are optional. 

A multi_burst is simply a collectino of `ind_bursts`. The `ind_burst` is where the actual data is stored, and can be modified at the row level. Labels are also created at the `ind_burst` level and created based on the active_burst. We'll discuss the use of these labels later, but they are largely internal but can be used to quickly subset data from a burst.

#### Basics

##### ind_bursts 
An ind_burst is the grouping variables for a single row of data.  

You can make an `ind_burst` object using `make_ind_burst()`, and giving it a list with the burst variables named. In this example we have a single sensor named 'CJ13' from a raccon, and an additional grouping varaiable of month (entered as its numeric interpretation).

ind_bursts convert these variables into characters and creates a label from the active_burst by paste0(sep = '_').

```{r}

indb <- make_ind_burst(list(id='CJ13', month = 4), active_burst = c('id','month'))
str(indb)
```

Because `ind_burst`s are simply lists, you can edit individual elements in an ind_burst
```{r}
indb 
indb[1] <- 'CJ15'
indb$month <- '5'
str(indb)
```

#### Active burst of the ind_burst class

`ind_burst`s is a grouping class where you can have different groups 'turned on' (aka activated) as the analysis allows. Which groups are activated is determined by the 'active burst'. You can set this when creating an ind_burst. If no active_burst is given it defaults to all grouping names being the active_burst. 

```{r}
indb <- make_ind_burst(list(id='CJ13', month = 4), active_burst = c('id'))
str(indb)
indb <- make_ind_burst(list(id='CJ13', month = 4))
str(indb)

````

You can access or change the `active burst` at anytime using the `active_burst()` function. The active burst tells any analysis to group by that mix of variables. 

```{r}
active_burst(indb)
active_burst(indb) <- 'id'
str(indb)

```

To speed up this process by calculating a 'label' attribute is calculated for each ind_burst which is a combination of the active burst variables. This label is to increase calculation and subsetting speed, and is therefore not available for renaming. It can be accessed however by `burst_labels`. This can be useful when refering to a burst or selecting many bursts in a column

```{r}
burst_labels(indb)
burst_labels(mb1)[1:10]
```

##### multi_burst  
Multi_bursts are a collection of ind_bursts where all ind_bursts have the same active_burst.

Similarly to ind_burst you can make a multi_burst with `make_multi_burst()`. The argument `burst` takes a list where each element is a vector indicating the named burst as well as a vector of the active bursts.

```{r}
burst_list <- list(id = rep(1:2,10), year = rep(2020, 10))
mb <- make_multi_burst(x=burst_list, active_burst=c('id','year'))
str(mb)
```

You can also make a multi_burst by concatenating multiple ind_bursts. All `ind_bursts` must have the same active_burst otherwise an error is returned

```{r}
a <- make_ind_burst(list(id = 1, year = 2020))
b <- make_ind_burst(list(id = 1, year = 2021))
c <- make_ind_burst(list(id = 2, year = 2020))
mb <- c(a, b , c)

summary(mb)
```

You can also combine multi_bursts together with `c()`.  

```{r}
mb_combine <- c(mb,mb)
summary(mb_combine)
```
You can also edit bursts like a list, but you must replace it with an object of the appropriate class and active_burst
```{r}
mb[1]
mb[1] <- make_ind_burst(list(id=3,year=2019))
mb[1]

```

And the burst names must match the ones in the multi_burst
```{r}
# Try to add an ind_burst with a month field when the original burst had year instead 
try( mb[1] <- make_ind_burst(list(id=3,month=2019)) )
```

##### selecting a burst

As `multi_burst`s are stored as lists, it can be difficult to refer to a single burst or group of bursts. This is where the burst label can come in handy. The burst label is remade everytime an active burst changes, and therefore can be used to subset. Using `burst_label()` you can access the variable to subset with. Additionally the label comes with a factor argument that returns a factor instead of a character. But it should be noted the variable is stored internally as a character in each ind_burst and the factor is calculated later only when called, it is not stored. This functionality is largely for internal use, but may be of interest for people wanting speed up analysis.

```{r}
burst_list <- list(id = rep(1:2,10), year = rep(2020, 10))
mb <- make_multi_burst(x=burst_list, active_burst=c('id','year'))
burst_labels(mb)[1:10]

# Subsetting a particular sensor from our raccoon data

data <- read.csv(system.file('extdata/raccoon_data.csv', package='sftrack'))
data$month <- as.POSIXlt(data$acquisition_time)$mon+1

data$time <- as.POSIXct(data$acquisition_time, tz='EST')
coords = c('longitude','latitude')
burst = list(id = data$sensor_code, month = as.POSIXlt(data$acquisition_time)$mon+1)
time = 'time'

my_sftraj <- as_sftraj(data = data, coords = coords, burst = burst, time = time)
head(my_sftraj[burst_labels(my_sftraj) %in% c('CJ11_1'), ])


```

You can also subset by entering the burst label of the burst itself in either the multi_burst or the sftrack/sftraj object:

```{r}
head(mb['1_2020'])

sub <- my_sftraj['CJ13_1',]
print(sub,5, 3)
```
##### active_burst 
The active_burst is a simple yet powerful feature. It dictates how your data is grouped for essentially all calculations. It can also be changed on the fly. You can view and change the active_burst of a multi_burst with `active_burst()`. Once changed, it changes the active_burst for all `ind_burst` and recalculates their labels.

Active_bursts can be changed for any sftrack/straj/multi_burst/ind_burst

```{r}
# sftrack
active_burst(my_sftraj)
summary(my_sftraj, stats = T)
active_burst(my_sftraj) <- c('id')
active_burst(my_sftraj)
summary(my_sftraj, stats = T)

# multi_burst
active_burst(mb)
active_burst(mb) <- 'id'
active_burst(mb)

# ind_burst
active_burst(indb)
active_burst(indb) <- 'id'
active_burst(indb)

```