---
title: "Reproducible Research: Peer Assessment 1"
output: 
  html_document:
    keep_md: true
---


## Loading and preprocessing the data

Before trying to read in the data, it first needed to be unzipped.

```r
if(!file.exists("activity.csv")){
  unzip("activity.zip")
}
```

The csv file was read in using the following code.

```r
stepData <- read.csv("activity.csv")
```

The dates were initially read in as factors.

```r
str(stepData)
```

```
## 'data.frame':	17568 obs. of  3 variables:
##  $ steps   : int  NA NA NA NA NA NA NA NA NA NA ...
##  $ date    : Factor w/ 61 levels "2012-10-01","2012-10-02",..: 1 1 1 1 1 1 1 1 1 1 ...
##  $ interval: int  0 5 10 15 20 25 30 35 40 45 ...
```

So they needed to be converted into dates.

```r
library(dplyr)
```

```
## Warning: package 'dplyr' was built under R version 3.6.3
```

```r
library(lubridate)
```

```
## Warning: package 'lubridate' was built under R version 3.6.3
```

```r
stepData <- mutate(stepData, date = ymd(as.character(stepData$date)))
str(stepData)
```

```
## 'data.frame':	17568 obs. of  3 variables:
##  $ steps   : int  NA NA NA NA NA NA NA NA NA NA ...
##  $ date    : Date, format: "2012-10-01" "2012-10-01" ...
##  $ interval: int  0 5 10 15 20 25 30 35 40 45 ...
```

## What is mean total number of steps taken per day?

First, the daily step totals were calculated.

```r
#Calculating daily totals
dailySteps <- aggregate(stepData$steps, list(stepData$date), sum)

#Renaming dataframe columns
names(dailySteps) <- c("date","steps")
```

Then a histogram of the daily step totals was created.

```r
hist(dailySteps$steps, main="Histogram of Total Number of Steps Taken in a Day", xlab="Total number of steps")
```

![](PA1_template_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

Finally, the mean and median values of the total number of daily steps were calculated.

```r
#Calculating the mean
mean(dailySteps$steps, na.rm=TRUE)
```

```
## [1] 10766.19
```

```r
#Calculating the median
median(dailySteps$steps, na.rm=TRUE)
```

```
## [1] 10765
```

## What is the average daily activity pattern?

First, the average number of steps taken in each interval were calculated.

```r
#Calculating average number of steps taken in each interval
intervalAvg <- aggregate(stepData$steps, list(stepData$interval), mean, na.rm=TRUE)

#Renaming dataframe columns
names(intervalAvg) <- c("interval", "avgSteps")
```

As the interval values are represented by using the last 2 digits to show the minute and the leading digits to show the hour, the interval values had to be converted to a time value that would be suitable to plot, so a new variable (called time) was created that represented each time interval as the number of hours since midnight.

```r
#Calculting time at each interval (as number of seconds since midnight)
times <- as.numeric(hm(paste(as.character(intervalAvg$interval %/% 100), as.character(intervalAvg$interval %% 100))))
intervalAvg <- mutate(intervalAvg, time=times)

#Converting time from number of seconds since midnight to number of hours since midnight
intervalAvg <- mutate(intervalAvg, time=time/60/60)
```

The average number of steps was then plotted against this new time variable.

```r
library(ggplot2)
```

```
## Warning: package 'ggplot2' was built under R version 3.6.3
```

```r
ggplot(intervalAvg, aes(x=time, y=avgSteps)) + geom_line() + scale_x_continuous(breaks=seq(0,24,1)) + ylab("Average number of steps during interval") + xlab("Time (Hours)") + ggtitle("The Average Number of Steps Taken at Each Time Interval")
```

![](PA1_template_files/figure-html/unnamed-chunk-10-1.png)<!-- -->

And the interval with highest average number of steps was calculated.

```r
intervalAvg[which.max(intervalAvg$avgSteps), "interval"]
```

```
## [1] 835
```

## Imputing missing values

First the number of rows that conatined any NAs was calculated

```r
sum(!complete.cases(stepData))
```

```
## [1] 2304
```

Then the number of NA values in each column were calculated to see which columns contained NA values.

```r
#Calculating number of NA values in steps
sum(is.na(stepData$steps))
```

```
## [1] 2304
```

```r
#Calculating number of NA values in date
sum(is.na(stepData$date))
```

```
## [1] 0
```

```r
#Calculating number of NA values in interval
sum(is.na(stepData$interval))
```

```
## [1] 0
```

The steps variable was the only one with any NA values, and so a new dataframe was created with these NA values imputed (The NA values were replaced with the average step value for the time interval):

```r
#Creating function that returns the average
avgForInterval <- function(interval, intervalAvg){
  intervalAvg$avgSteps[intervalAvg$interval == interval]
}

#Creating a new dataframe (stepDataImputed) with all NA values replaced with the average step value for that interval
stepDataImputed <- stepData
stepDataImputed[is.na(stepDataImputed$steps),]$steps <- sapply(stepDataImputed[is.na(stepDataImputed$steps),]$interval, avgForInterval, intervalAvg)
```

Then a histogram of the total number of steps per day was created using the data frame with the imputed values.

```r
#Calculating daily totals from the imputed dataframe
daySums <- aggregate(stepDataImputed$steps, list(stepDataImputed$date), sum)

#Renaming dataframe columns
names(daySums) <- c("date", "totalSteps")

#Creating histogram 
hist(daySums$totalSteps, main="Histogram of Total Number of Steps Taken in a Day (Imputed Data)", xlab="Total number of steps")
```

![](PA1_template_files/figure-html/unnamed-chunk-15-1.png)<!-- -->

Finally, the mean and median values of the total number of daily steps including imputed values were calculated.

```r
#Calculating the mean
mean(daySums$totalSteps, na.rm=TRUE)
```

```
## [1] 10766.19
```

```r
#Calculating the median
median(daySums$totalSteps, na.rm=TRUE)
```

```
## [1] 10766.19
```

When this histogram is compared to the histogram of the number of daily steps without the imputed data, you can see that the total frequency has increased by 8. This means that 8 days had entirely NA values, and there were no other NA values except on these days as 8 * 288(the number of intervals per day) = 2034(the number of missing values in the dataset). As such, it is unsurprising that these new days have all been added to the middle bucket of the histogram (as this is where the mean daily total is). The mean of the daily total os steps also remained unchanged, and the median value is now equal to the mean.

## Are there differences in activity patterns between weekdays and weekends?

First, a new factor was added to indicate whether each observation was taken on a weekday or on the weekend.

```r
#Creating function that assigns a given day to be either a weekend or a weekday
weekend <- function(dayNum){
  if(dayNum==1 | dayNum==7){
    return("weekend")
  }
  else if(dayNum>1 & dayNum<7){
    return("weekday")
  }
  else{
    return(NA)
  }
}

#Applying function to get a list of wether each observation was taken on a weekday or weekend
weekend <- sapply(wday(stepDataImputed$date), weekend)

#Adding new factor variable to each row of the imputed dataframe to indicated wether it was taken on a weekday or weekend
stepDataImputed <- mutate(stepDataImputed, wEnd=factor(weekend, levels=c("weekday","weekend")))
```

Time series plots were then created to show the average number of steps taken at each interval across both weekdays and weekends.

```r
#Calculating the average number of steps in each interval on both weekdays and weekends
avgStepsByWeekend <- stepDataImputed %>% group_by(wEnd, interval) %>% summarize(avgSteps = mean(steps))

#Calculting time at each interval (as number of seconds since midnight)
avgStepsByWeekend <- mutate(avgStepsByWeekend, time=times)

#Converting time from number of seconds since midnight to number of hours since midnight
avgStepsByWeekend <- mutate(avgStepsByWeekend, time=time/60/60)

#Plotting time series graph of the average number of steps against the time variable, with two panels to show how this varied between weekdays and weekends.
ggplot(avgStepsByWeekend, aes(x=time, y=avgSteps)) + geom_line() + facet_wrap(. ~ wEnd) + scale_x_continuous(breaks=seq(0,24,2)) + ylab("Average number of steps during interval") + xlab("Time (Hours)") + ggtitle("The Average Number of Steps Taken at Each Time Interval")
```

![](PA1_template_files/figure-html/unnamed-chunk-18-1.png)<!-- -->
