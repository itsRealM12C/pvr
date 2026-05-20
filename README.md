# pvr
Extract the image in the PVR file.

## Work
While reverse engineering the file, i expected file to be B&W, but it's actually ETC1, which ETC1 is full RGB. It stores color base values per 4×4 block and applies luminance modifiers on top, so you get proper color out of it.
