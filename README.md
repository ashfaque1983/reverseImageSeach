# Reverse Image Search for Drupal 10.3

A custom Drupal module that implements vector-based reverse image search for Media Library without third-party dependencies.

## Features

- Search Media Library images by uploading a similar image
- Pure PHP implementation with no external dependencies or APIs
- Uses advanced image recognition techniques:
  - Perceptual Hash (pHash)
  - Color Histogram analysis
  - Edge/Shape detection
- Configurable similarity thresholds and feature weights
- Responsive UI with visual similarity indicators
- Compatible with Drupal 10.3+

## Requirements

- Drupal 10.3 or higher
- PHP 8.1 or higher with GD extension
- Drupal Media module enabled

## Installation

1. Extract the `reverse_image_search.zip` file to your Drupal site's `modules/custom` directory
   ```
   cd [drupal-root]/modules/custom
   unzip reverse_image_search.zip -d reverse_image_search
   ```

2. Enable the module through the Drupal admin interface
   - Go to `Admin > Extend`
   - Find "Reverse Image Search" in the module list
   - Check the box and click "Install"

3. Set permissions
   - Go to `Admin > People > Permissions`
   - Grant "Access reverse image search" to appropriate roles
   - Grant "Administer reverse image search" to administrator roles

## Configuration

1. Go to `Admin > Configuration > Media > Reverse Image Search`
2. Adjust settings:
   - **Results per page**: Number of search results to display (default: 20)
   - **Default similarity threshold**: Minimum similarity score for results (default: 60%)
   - **Default feature weights**: Adjust importance of different image features
     - Color Weight (default: 40%)
     - Edge/Shape Weight (default: 30%)
     - Perceptual Hash Weight (default: 30%)
   - **Advanced settings**:
     - Image Processing Dimensions: Size to resize images to before processing (default: 256px)
     - Color Histogram Bins: Number of bins for color analysis (default: 32)

3. Index existing media
   - The module automatically starts indexing your Media Library images upon installation
   - You can monitor the indexing progress on the settings page
   - You can manually rebuild the index using the "Rebuild Index" button

## Usage

1. Go to `Admin > Content > Reverse Image Search`
2. Upload an image to search for similar images in your Media Library
3. Adjust search parameters (optional):
   - Similarity threshold
   - Feature weights
4. Click "Search" to find similar images
5. Results are displayed with similarity scores and visual indicators
6. Click on a result to view the full media item

## How It Works

The module uses multiple image processing techniques to find similar images:

1. **Perceptual Hash (pHash)**: Creates a "fingerprint" of the image that is resilient to scaling, compression, and minor edits
2. **Color Histogram**: Analyzes the distribution of colors in the image
3. **Edge Detection**: Identifies shapes and structures using Sobel operators
4. **Vector Comparison**: Compares feature vectors using cosine similarity and other metrics

These features are extracted from Media Library images and stored in a database. When you upload a query image, the same features are extracted and compared to find matches.

## Technical Implementation

- Custom database table for storing image feature vectors
- Drupal Queue API for batch processing large media libraries
- OOP architecture with separate image processing classes
- Built-in pagination for browsing results
- AJAX support for dynamic updates

## Troubleshooting

If you encounter issues:

1. **Missing GD extension**: Ensure PHP GD extension is installed and enabled
2. **Memory issues**: For large images, you may need to increase PHP memory limit
3. **Performance concerns**: Adjust the advanced settings for a balance of accuracy vs. speed
4. **Empty results**: Check that Media Library images are properly indexed

## Credits

Developed by Ashfaque Ahmed

## License

This module is licensed under GPL v2 or later.
