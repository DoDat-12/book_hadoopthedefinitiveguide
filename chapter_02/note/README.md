# Chapter 2. MapReduce

## A weather Dataset

### Data Format

The data we will use is from the **National Climatic Data Center**, or NCDC. The data is stored using a line-oriented
ASCII format, in which each line is a record.

For simplicity, we focus on the basic elements, such as temperature, which are always present and are of fixed width.

## Analyzing the Data with Unix Tools

Script to calculate the maximum temperature for each year (42 mins to run all lines and files)

    #! /usr/bin/env bash
    for year in all/*
    do
        echo -ne `basename $year .gz`"\t"
        gunzip -c $year | \
            awk '{  temp = substr($0, 88, 5) + 0;  # temperature, add 0 to turn into integer
                    q = substr($0, 93, 1);  # quality
                    if (temp != 9999 && q ~ /[01459]/ && temp > max) max = temp }
                END { print max }'
    done

To speed up the processing, we need to run parts of the program in parallel. In theory: process different years in
different processes, using all the available hardware threads on a machine. There are a few problems with this:

- Diving the work into equal-size pieces isn't easy. In this case, the file size for different years varies widely, so
  some processes will finish much faster than others. Better approach: Split the input into fixed-size chunks and assign
  each chunk to a process.

- Combining the results from independent processes may require further processing. If using the fixed-size chunk
  approach, the combination is more delicate. For this example, data for a particular year will typically be split into
  several chunks, each processed independently. We'll end up with the maximum temperature for each chunk, so the final
  step is to look for the highest of these maximums for each year.

- You are limited by the processing capacity of a single machine.

Although it's feasible to parallelize the processing, in practice it's messy. Using a framework like Hadoop to take care
of these issues is a great help.

## Analyzing the Data with Hadoop

### Map and Reduce

MapReduce works by breaking the processing into two phases:

- The map phase
- The reduce phase

Each phase has key-value pairs as input and output, the types of which may be chosen by the programmer.

#### Map Phase

The input to our map phase is the raw NCDC data. We choose a text input format that gives us each line in the dataset as
a text value. The key is the offset of the beginning of the line from the beginning of the file, but as we have no need
for this, we ignore it.

Map function: Pull out the year and the air temperature. In this case, the map function is just a data preparation
phase, setting up the data for the reduce function to find the maximum temperature for each year. (drop bad records too)

Sample lines of input data:

    0067011990999991950051507004...9999999N9+00001+99999999999...
    0043011990999991950051512004...9999999N9+00221+99999999999...
    0043011990999991950051518004...9999999N9-00111+99999999999...
    0043012650999991949032412004...0500001N9+01111+99999999999...
    0043012650999991949032418004...0500001N9+00781+99999999999...

As key-value pairs: (the keys are the line offset within the file)

    (0, 0067011990999991950051507004...9999999N9+00001+99999999999...)
    (106, 0043011990999991950051512004...9999999N9+00221+99999999999...)
    (212, 0043011990999991950051518004...9999999N9-00111+99999999999...)
    (318, 0043012650999991949032412004...0500001N9+01111+99999999999...)
    (424, 0043012650999991949032418004...0500001N9+00781+99999999999...)

THe map function merely extracts the year and the air temperature:

    (1950, 0)
    (1950, 22)
    (1950, -11)
    (1959, 111)
    (1949, 78)

The output from the map function is processed by the MapReduce framework before being sent to the reduce function. This
processing sorts and groups the key-value pairs by key. So, our reduce function sees the following input:

    (1949, [111, 78])
    (1950, [0, 22, -11])

All the reduce function has to do is iterate through the list and pick up the maximum reading

    (1949, 111)
    (1950, 22)

![MapReduce logical data flow.png](MapReduce%20logical%20data%20flow.png)

A `Job` object forms the specification of the job and gives you control over how the job is run. When we run this job on
a Hadoop cluster, we will package the code into a JAR file (which Hadoop will distribute around the cluster)
