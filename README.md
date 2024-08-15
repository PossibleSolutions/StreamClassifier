# StreamClassifier
Takes habitat/object input with geometry (streams in line format) and classifies (naturality assesment) them to stream or ditch like features. Classification is done with Random Forest and Neural Networks. This is used in Part2 to divide a raster mask into drained and natural parts.


## General pipeline
Part1
- Reading input, dissolving it by group and subsetting it into smaller sets for processing
- Creation of classification context
- Application of machine learning methods to classify the data
- Validation metrics
- Adding data from other yearly (topographic database) versions
- Assessment of variable significance
- Adding reference values from previous versions
  
Part2 
- Class separation
- Buffering
- Rasterization
- Addition of additional data, fields and roads as ditches
- Mask comparison and addition
- File compression

## Inputs
- NLS Topographic database (https://www.maanmittauslaitos.fi/en/maps-and-spatial-data/datasets-and-interfaces/product-descriptions/topographic-database) “kohdeluokka” 36311 (stream, under 2m) and “kohdeluokka” 36312 (stream 2-5m).
- Training material, where streams are classified manually (or in some other valid method) into “natural streams” and “ditches”.
- Additional data, e.g. fields and roads as ditches.
- Background mask to divide into drained/natural parts, petlands and mineral soils of Finland.

## Outputs
- Input features with added variables used in classification, including length, simplified length, ratio of these lengths, number of vertices, sinuosity, sum and mean distance to neighbor line centroids, max and mean distance in vertices, depth to water raster values, slope raster values, soil raster values and predicted class. predicted class in scale 0.00 – 1.00, where 1 is natural stream like and 0 ditch like.
- Input mask raster (with peatlands and mineral soils) which is divided into drained and natural parts.

## Overview
#### Reading input, dissolving it by group and subsetting it into smaller sets for processing
- dataset is grouped by "kohdeluokka" which tells about the size of the stream (under 2m and 2-5m classes) 
- each group is dissolved separately, so that the lines are less fragmented 
- dask geopandas is used to partition the input data into npartitions = input. This is due to heavy processing cost and need to process in steps 


#### Creation of classification context
Each input row is processed to include additional variables that could explain the class differences.
- define input directory to loop through, usually the folder defined in the previous step, including the input in parts
- define reference data with manually classified features. this is used to extract input features that intersect with reference input as reference data. enables data homogeneity, if reference data is in a different form
- create a "counter system" that enables stopping the process midway. On the next run remembers the last processed part and continues from there. this requires that the user does not add extra files into output folder
- calculates geometry-attributes of each input line: length, simplified length, ratio of these lengths, number of vertices, sinuosity, sum and mean distance to neighbor line centroids, max and mean distance in vertices
- samples raster values in each vertex and calculates mean and median values for each line. DTW (depth to water), slope, soil type


#### Append the data that was processed in parts into one file
For practical purposes and eg. when the data is normalized and global minmax values required. In some cases training within subsets can be beneficial (as criterium might differ in different subsets) and appending is not suggested. 


#### Drop geometric duplicates
Spatial joins might create duplicates. Drop them here if necessary.


#### Apply machine learning methods to classify the data
For this supervised learning, reference data with ref_class variable is needed.
- define reference for model training
- fill missing data, fill outliers utilizing z-score, normalize for both reference data and classifiable data
- train model
- classify probability with random forest and validate with accuracy metrics and AUC and ROC-curve
- classify probability with neural networks
- combine classification probability from both random forest and neural network mean


#### Add data from other yearly versions
to cover more area, eg. restoration of peatlands can lead to filled streams and removed lines. In this classification once ditched, is always ditched
-for this additional data, only "Creation of classification context" and "Apply machine learning methods" are run, with the assumption that as only additional data, there is no need to dissolve (disjoint parts are short) and that it is not necessary to split it in smaller processing parts

#### Asses variable significance
All variables are not necessarily significant in classification and can cause misclassification.
- shapley values tell about variable significance
- beware. computationally heavy and needs the classifier block to be run first eg. classifier = RandomForestClassifier(n_estimators=20,criterion='gini', random_state= 0, n_jobs=-1)
- these results can be used to optimize the model via variable selection


#### Still not good? Consider following…
#### Add more training data
Pseudo reference data was added, where the model uses a filtered dataset as input, instead of manually classified one. Adds human error and bias to the model as all filter="vertices=2,sinuosity=1,NeighborDistMean<75" are not in fact ditches
#### Add more variables
E.g. here sequential distances were added (distances between vertices instead of whole line).



This resulted into input vector geometries (streams) with added variables and predicted class (stream or ditch) and its probability. You can find this part in the repo with name StreamClassification_pipeline_Github.ipynb. This output was further utilized in creation of a raster, where a raster mask with peatlands and mineral soils were further divided into drained and natural parts. You can find this part in the repo with name StreamClassification_MaskComparison_Github.ipynb.

### Part2

#### Divide into x classes
- stream, ditch 
- combine them so that the emphasized option overwrites the other if overlapping
- if one wants a more streamlined categorized inspection instead of a continuous probabilistic one 
- e.g. 
stream = df[
    (df['num_vertices'] > 6) &
    (df['sinuosity'] > 1.02) &
    (df['mean_distance_seq'] < 10) &
    ((df['StreamProbRF'] > 0.6) | (df['StreamProbNN'] > 0.6))]


#### Buffer creation
50 meters was assumed to be the impact area of a ditch, so this 50-meter buffered line geometry can be considered the drained area.

#### Rasterization
If the mask is a raster file, then the vector geometries need to be rasterized.

#### Addition of 50m buffered fields and roads as ditches
Fields and roads are most commonly ditched, so they are defined as drained area and added to the stream raster. Stream_raster + FieldRoad_raster

#### Mask analysis
The raster mask with peatlands and mineral soils was further divided into drained and natural parts by doing a simple addition stream_raster + mask_raster.  Both rasters must have the same properties (resolution, extent, crs…), if in your case not, resample first.

The ditch/stream raster had two categorical values (multiplied by 10), 20 = peatland, 10 = mineral soil. The stream raster had values 3 = drained area, 4 = natural stream, thus the possible sum outcomes for stream raster + mask were following:
- 0 = no data
- 3 = drained area not intersecting mask 
- 4 = natural stream not intersecting mask 
- 10 = mineral soil not intersecting drained/stream area 
- 13 = mineral soil intersecting drained area 
- 14 = mineral soil intersecting stream area 
- 20 = peatland not intersecting drained/stream area 
- 23 = peatland intersecting drained area 
- 24 = peatland intersecting stream area 

#### Compress end product for storage
Raster files can be quite large, so compression for storage is often recommended. LZW compression with SPARSE_OK was utilized.

