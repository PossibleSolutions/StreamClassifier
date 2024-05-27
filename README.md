# StreamClassifier
Takes habitat/object input with geometry (streams in line format) and classifies (naturality assesment) them to stream or ditch like features. Classification is done with Random Forest and Neural Networks


General pipeline
- Reading input, dissolving it by group and subsetting it into smaller sets for processing
- Creation of classification context
- Application of machine learning methods to classify the data
- Validation metrics
- Adding data from other yearly (topographic database) versions
- Assessment of variable significance
- Adding reference values from previous versions

Inputs
- NLS Topographic database (https://www.maanmittauslaitos.fi/en/maps-and-spatial-data/datasets-and-interfaces/product-descriptions/topographic-database) “kohdeluokka” 36311 (stream, under 2m) and “kohdeluokka” 36312 (stream 2-5m).
- Training material, where streams are classified manually (or in some other valid method) into “natural streams” and “ditches”.

Outputs
- Input features with added variables used in classification, including length, simplified length, ratio of these lengths, number of vertices, sinuosity, sum and mean distance to neighbor line centroids, max and mean distance in vertices, depth to water raster values, slope raster values, soil raster values and predicted class. predicted class in scale 0 – 1, where 1 is natural stream like and 0 ditch like.
