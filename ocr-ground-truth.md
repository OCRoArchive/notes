# Creating Ground Truth for OCR Training

(TL;DR) The simple answer is: if you want to create usable OCR training data, first follow the instructions in the very next section: for each image file like `page1.png`, create a corresponding `page1.gt.txt` file containing the plain text contained in the image in reading order, respecting line breaks. Do that for all images and tar it up in a tar file. We can use that training data directly. See below for some more details.

# Detailed Instructions

## Page Images with Page Transcriptions
 
This is probably what you should focus on. The process for creating these is fairly simple:

 - for each book/document, create a directory
 - within each directory, create sequentially numbered image files corresponding to pages; use PNG or JPEG compression (quality>=98) with extensions ".png" or ".jpg"
 - if you control the scanning, 600 dpi color or grayscale is the preferred resolution
 - for each page, type the entire text on the page in reading order into a text file with the extension ".gt.txt", respecting line breaks
 
Some details for the transcription:

 - line breaks should be exactly as they are on the page
 - use unicode UTF-8 encoding
 - decide on a character set you want to use and only transcribe characters in that character set
 - you should enter headers, footers, page numbers, and footnotes as written
 - keep your document valid Markdown (but don't use Markdown markup)
 - use HTML entities consistently for characters that have special meaning in Markdown (&amp; &lt; &gt; &#35; &36;), and for backslash
 - you can use HTML entities for any characters that are difficult to type on a keyboard
 - do not use PDF or TIFF to encapsulate images

The above is sufficient for most training; anything that can't be transcribed (math, chemical formulas, etc.), just leave out.

Some additional details:

 - treat captions as if they were separate text blocks/paragraphs
 - transcribe superscripts/subscripts (for footnotes) using their Unicode codepoints (not using Markdown syntax)
 - if you choose to transcribe math or chemical formulas, use Markdown syntax (`$...$`, `$\ce{...}$`)
 - indicate wide spaces / tabulation by using the TAB character (both in headers and tables)
 
After you have transcribed the data, create a .tar archive. For books, it's often convenient to have one .tar archive per book. For smaller documents, you can have multiple documents per archive. A good target size for page collections is 100 Mbytes to 10 Gbytes. The files in such an archive in sequence look something like this:

```
    mobydick/0001.png
    mobydick/0001.gt.txt
    mobydick/0002.png
    mobydick/0002.gt.txt
    ...
    alice/0001.png
    alice/0001.gt.txt
    alice/0002.png
    alice/0002.gt.txt
    ...
    canterburytales/0001.png
    ...
```

If you have both cropped/deskewed and uncropped versions, or both grayscale and binarized versions, provide them like this:

```
    mobydick/0001.raw.png
    mobydick/0001.deskewed.png
    mobydick/0001.gt.txt
    ...
```


The only thing that OCR training depends on is that images and ground truth are in adjacent files and have the same prefix up to the first "." in the filename. The rest of the directory/file naming structure helps humans to make sense of the data and is used by a few tools to help with visualization.

Since these .tar archives may be unpacked on disk by users, it's a good idea to stick to only letters, digits, and underscores in the file names and to ensure that book/document names are unique across the entire collection of documents (so that all tar files can be extracted in the same directory). You can use arbitrary directory trees in the prefixes to help organize the documents.

If you want to add metadata or other kind of information, you can put it into a file called __README__ (human readable text) or __JSONINFO__ (JSON data) at the very beginning.

## Untranscribed Page Scans

This is useful for unsupervised and self-supervised training. The more data you can provide in this form, the better. As above, JPEG or PNG format images at 600 dpi in grayscale or color are preferred. Any such data can just be tarred up in a tar file with whatever 

## Pages with OCR Output

You can OCR in two different ways:

 - use it as a starting point for manual transcription
 - provide it "as is" as training data for other engines
 
As a starting point for manual transcription, you can either use an OCR correction tool, or you can just run the OCR engine from the command line and compare the text to the image. 

There are many tools available supporting both correction and batch-OCR, and you should probably use such a tool. Just for illustration purposes, let's see how you might carry out these steps using `tesseract` as the OCR engine at the command line for a single file.

```
    $ tesseract page1.png page1 -l eng
    $ display page1.png &
    $ vi page1.txt # perform corrections
    $ mv page1.txt page1.gt.txt
```

If you have the output from previous OCR runs, they can be usefully used for training new OCR systems. Follow the same .tar file structure as above, replacing `.gt.txt` files with OCR output files.

For Tesseract, this might look like this (for each file):

```
    $ mv page1.png page1.raw.png
    $ deskew page1.raw.png -o page1.deskewed.png
    $ binarize page1.deskewed.png -o page1.binarized.png
    $ tesseract page1.binarized.png page1 -l eng hocr
```

The output from this kind of processing looks like this:

```
    mobydick/0001.raw.png
    mobydick/0001.deskewed.png
    mobydick/0001.binarized.png
    mobydick/0001.hocr
    ...
```

Providing output in `.hocr` format is preferred because that's what the current tools understand, it's easy to parse, and it contains all the information needed for OCR training. If you have some other OCR output (e.g., Abbyy XML), just provide that with the appropriate extension. It is helpful to provide human readable information in a __README__ at the start of every .tar file.

It's generally best to run OCR engines on deskewed images, and, depending on the OCR system, to binarize before. If you perform such preprocessing, include the raw, deskewed, and binarized data along with the OCR output.

## Segmentation Training Data

When training OCR systems for difficult scripts or handwriting, having manual segmentation information is helpful. A few tens of thousands of lines of manual segmentation is also needed for bootstrapping from real data if no other OCR engines or training data are available. Such segmentation information either marks every "word" separately, or every text line; the decision of what segments to mark depends on the script and language.

There are several different ways of providing this kind of information:

 - axis aligned bounding boxes: easy to enter with a mouse, but only works for deskewed images and some scripts
 - polygonal bounds for words/text lines: works on most scripts and skewed pages, but may take a long time to enter
 
The following are image-based layout markups that can be easily created with any image editor or pen based sketching app (but you can also develop special tools for them).

 - hand-drawn baseline: manually draw an approximation to the baseline of each text line with a pen; baselines of separate lines must not touch
 - hand-drawn center line: manually draw a line through the center of each text line with a pen; center lines of separate lines must not touch
 
Baselines and centerlines are easy to enter and can be converted to bounding boxes or polygons automatically. To create them easily in a drawing program, follow these steps:

 - convert the page images to grayscale (this is just for human reference)
 - load each page image into the image editor
 - with a red pen that is at least 3 pixels tall and wide, and without anti-aliasing, draw lines through the center of each text line directly on the grayscale image
 - save the result as a color PNG files with an extension of `.centerlines.png`
 
For the most complex segmentation tasks, you can take the following approach:

 - in an image editor, load the original page image into one layer in grayscale
 - in a separate layer, completely paint each region corresponding to a separate word or line (depending on what you are trying to mark up); be sure to use a "hard pen" (no anti-aliasing)
 - it's helpful to use saturated, distinct primary colors for the labels
 - you can reuse colors provided distinct segments that touch don't use the same color
 - you can set the segmentation layer to partially transparent to see the underlying image
 - when all done, save just the color layer as `.colseglines.png` or `.colsegwords.png`
 
 Note that you can also generate `.colseglines.png` and `.colsegwords.png` images directly from OCR outputs (e.g. by rendering polygonal outputs).

## Semantic Layout Training Data

For training data for semantic layout analysis, you can generate training data in a way similar to the color-based segmentation above:

 - first, pick a set of colors representing different semantic regions; e.g. "body text" = "#ff0000", "section headings" = "#ff7f00", "image/figure" = "#00ff00", "other text" = "#0000ff", "noise/marginal noise" = "#7f7f7f", "table" = "#007f7f"
 - document the assignments in a __README__ file
 - convert the page images to grayscale (this is just for human reference)
 - load each page image into the image editor
 - paint each region in the image with the color corresponding to its semantic meaning
 - save the color image layer as `.colsemseg.png`
 
 