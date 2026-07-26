# prep_for_social.py
"""
Adding watermark and prepping photos to lower resolution to release on socials
Batch-processes a folder of photos for safe social media posting:
  1. Resizes down to a max dimension (default 2048px on the long edge)
  2. Strips EXIF metadata (removes GPS location, camera info, etc.)
  3. Adds a semi-transparent text watermark in a corner
  4. Saves everything to an output folder, originals untouched

USAGE:
    python3 prep_for_social.py --input /path/to/originals --output /path/to/ready_to_post

OPTIONAL FLAGS:
    --watermark-text   "@uniandrainbowkitty"   (default text)
    --position         bottom-right | bottom-left | top-right | top-left  (default bottom-right)
    --max-size         2048   (long edge in pixels, default 2048)
    --opacity          128    (0-255, default 128 = ~50% transparent)
    --font-size-pct    3.0    (watermark text height as % of image's long edge, default 3.0)
    --quality          88     (JPEG quality 1-100, default 88)
    --logo             /path/to/logo.png   (use an image logo instead of text)

REQUIREMENTS:
    pip install Pillow --break-system-packages
"""



