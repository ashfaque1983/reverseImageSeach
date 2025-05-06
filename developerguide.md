# Reverse Image Search for Drupal 10.3 - Developer Documentation

This document provides technical details about the implementation of the Reverse Image Search module for Drupal 10.3.

## Architecture Overview

The module follows a layered architecture:

1. **Presentation Layer**: Forms, controllers, and templates for user interaction
2. **Service Layer**: Core business logic and coordination
3. **Image Processing Layer**: Feature extraction and comparison algorithms
4. **Data Storage Layer**: Database schema and query logic

### Directory Structure

```
reverse_image_search/
├── config/                      # Configuration definitions
├── css/                         # CSS stylesheets
├── js/                          # JavaScript files
├── src/                         # PHP source code
│   ├── Controller/              # HTTP request handlers
│   ├── Form/                    # Drupal forms
│   ├── ImageProcessor/          # Image analysis algorithms
│   └── Service/                 # Business logic services
├── templates/                   # Twig templates
└── [module files]               # Core module files
```

## Core Components

### Image Processing

#### 1. Perceptual Hash (PerceptualHash.php)

The pHash algorithm creates a hash that represents the visual appearance of an image, allowing for similarity comparison.

Implementation steps:
1. Resize image to 32x32 pixels
2. Convert to grayscale
3. Reduce to 8x8 for DCT (Discrete Cosine Transform)
4. Apply simplified DCT transform
5. Calculate the median value of DCT coefficients 
6. Generate a 64-bit hash based on whether each coefficient is above/below the median
7. Store as a 16-character hexadecimal string

Similarity is calculated using Hamming distance (number of differing bits).

#### 2. Color Histogram (ColorHistogram.php)

The color histogram captures the distribution of colors in an image.

Implementation steps:
1. Define a 3D histogram with configurable bins (default 32³)
2. Count pixels in each color bin (R,G,B dimensions)
3. Normalize the histogram (divide by total pixel count)
4. Flatten to 1D array for storage

Similarity is calculated using the Bhattacharyya coefficient.

#### 3. Edge Detection (EdgeDetection.php)

Edge detection identifies shapes and structures in the image.

Implementation steps:
1. Convert image to grayscale
2. Apply Sobel operators for edge detection
3. Calculate gradient magnitude at each pixel
4. Create a 16x16 grid representation of edge features
5. Store as a normalized vector

Similarity is calculated using cosine similarity.

#### 4. Vector Comparison (VectorComparison.php)

This utility class provides various vector similarity metrics:
- Cosine similarity
- Euclidean distance
- Manhattan distance
- Weighted score combination

### Service Layer

#### ReverseImageSearchService.php

This is the main service that coordinates:
1. Extracting features from images
2. Storing features in the database
3. Searching for similar images
4. Calculating combined similarity scores

Key methods:
- `indexMediaImage($media)`: Extract and store features for a media entity
- `searchSimilarImages($query_image_uri, $threshold, $weights, $limit, $offset)`: Find similar images
- `getIndexedImageCount()`: Get indexing statistics
- `clearIndex()`: Clear the feature database

### Data Storage

The module creates a custom database table `reverse_image_search_vectors` with the following schema:

```php
$schema['reverse_image_search_vectors'] = [
  'description' => 'Stores image feature vectors for Media Library items.',
  'fields' => [
    'id' => ['type' => 'serial', 'not null' => TRUE],
    'mid' => ['type' => 'int', 'not null' => TRUE],
    'fid' => ['type' => 'int', 'not null' => TRUE],
    'phash' => ['type' => 'varchar', 'length' => 64, 'not null' => TRUE],
    'color_histogram' => ['type' => 'blob', 'size' => 'big', 'not null' => TRUE],
    'edge_features' => ['type' => 'blob', 'size' => 'big', 'not null' => TRUE],
    'created' => ['type' => 'int', 'not null' => TRUE],
    'updated' => ['type' => 'int', 'not null' => TRUE],
  ],
  'primary key' => ['id'],
  'indexes' => [
    'media_id' => ['mid'],
    'file_id' => ['fid'],
    'phash' => ['phash'],
  ],
];
```

## Search Process

The search process follows these steps:

1. **Upload and validate** the query image
2. **Extract features** from the query image
3. **Compare** with stored features from Media Library
4. **Rank results** by similarity
5. **Apply threshold** to filter results
6. **Paginate** and display results

### Weighted Comparison Algorithm

The similarity score calculation combines multiple features:

```php
$similarity_score = 
  ($weights['phash'] * $phash_similarity) +
  ($weights['color'] * $color_similarity) +
  ($weights['edge'] * $edge_similarity);
```

Where:
- `$phash_similarity`: Normalized Hamming distance between perceptual hashes
- `$color_similarity`: Bhattacharyya coefficient between color histograms
- `$edge_similarity`: Cosine similarity between edge feature vectors

## Performance Considerations

### Memory Usage

Image processing can be memory-intensive. The module includes several optimizations:
- Images are resized before processing (configurable dimensions)
- Resources are freed after use
- Feature vectors are serialized for storage

### Indexing Strategy

The module uses Drupal's Queue API for indexing:
1. Media entities are queued during installation or index rebuild
2. A queue worker processes items during cron runs
3. Progress is tracked and displayed in the admin UI

### Query Optimization

For large media libraries, the module optimizes searches:
- Initial filtering by perceptual hash to reduce candidates
- Pagination to limit result set size
- Indexing on database fields for faster queries

## Extending the Module

### Adding New Feature Extractors

To add a new image feature extractor:

1. Create a new class in the `ImageProcessor` namespace
2. Implement extraction and comparison methods
3. Update `ImageFeatureExtractor::extractFeatures()` to include your new feature
4. Update `ReverseImageSearchService::compareFeatures()` to include your new comparison 
5. Add UI elements to allow configuring weight for the new feature

### Adding New Comparison Metrics

To add a new comparison algorithm:

1. Add the method to the `VectorComparison` class
2. Update the relevant feature extractor to use the new comparison
3. Update settings form if you want to make it configurable

## Troubleshooting

### Common Issues

1. **Memory limit exceeded**
   - Cause: Processing large images with insufficient PHP memory
   - Solution: Increase PHP memory_limit, reduce image processing dimensions

2. **Slow indexing**
   - Cause: Large media library with high-resolution images
   - Solution: Adjust queue processing parameters, optimize image dimensions

3. **Low-quality matches**
   - Cause: Feature weights not optimized for your content
   - Solution: Adjust feature weights to emphasize relevant characteristics

## Algorithm Details

### Perceptual Hash (pHash) Algorithm Pseudocode

```
function generateHash(image):
    # Resize and prepare
    img32 = resize(image, 32x32)
    img32 = convertToGrayscale(img32)
    img8 = resize(img32, 8x8)
    
    # Extract pixel values to 2D array
    pixels[8][8] = getPixelValues(img8)
    
    # Apply DCT
    dct = discreteCosineTransform(pixels)
    
    # Calculate average of DCT coefficients (skipping DC component)
    avg = average(dct[1:8][1:8])
    
    # Generate hash by comparing each coefficient to the average
    hash = ""
    for y = 0 to 7:
        for x = 0 to 7:
            if (x,y) != (0,0): # Skip DC coefficient
                if dct[y][x] > avg:
                    hash += "1"
                else:
                    hash += "0"
    
    # Return as hexadecimal
    return convertToHex(hash)
```

### Color Histogram Generation Pseudocode

```
function generateHistogram(image, numBins):
    width = imageWidth(image)
    height = imageHeight(image)
    
    # Initialize 3D histogram
    histogram[numBins][numBins][numBins] = zeros()
    
    # Count pixels
    totalPixels = 0
    for y = 0 to height-1:
        for x = 0 to width-1:
            r, g, b = getPixelColor(image, x, y)
            
            # Skip transparent pixels
            if isTransparent(image, x, y):
                continue
                
            # Map to bins
            rBin = floor(r * numBins / 256)
            gBin = floor(g * numBins / 256)
            bBin = floor(b * numBins / 256)
            
            histogram[rBin][gBin][bBin]++
            totalPixels++
    
    # Normalize
    for all bins:
        histogram[bin] /= totalPixels
    
    # Flatten to 1D array
    return flatten(histogram)
```

### Edge Detection Pseudocode

```
function detectEdges(image):
    # Create grayscale image
    grayscale = convertToGrayscale(image)
    
    # Define Sobel operators
    sobelX = [[-1, 0, 1], [-2, 0, 2], [-1, 0, 1]]
    sobelY = [[-1, -2, -1], [0, 0, 0], [1, 2, 1]]
    
    # Apply Sobel operators
    edgeMagnitude = zeros(width, height)
    for y = 1 to height-2:
        for x = 1 to width-2:
            # Apply convolution
            gx = convolve(grayscale, sobelX, x, y)
            gy = convolve(grayscale, sobelY, x, y)
            
            # Calculate gradient magnitude
            magnitude = sqrt(gx² + gy²)
            edgeMagnitude[y][x] = magnitude
    
    # Create feature grid (16x16)
    gridSize = 16
    features = []
    
    # Average edge magnitudes in each grid cell
    for each grid cell:
        cellAvg = averageEdgeMagnitudeInCell()
        features.append(normalizedCellAvg)
    
    return features
```

## Conclusion

This module provides a sophisticated yet efficient way to perform reverse image searches in Drupal's Media Library without external dependencies. By combining multiple image analysis techniques, it offers a robust solution that works well for a variety of visual content.
