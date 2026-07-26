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


mport argparse
import os
import sys
from pathlib import Path

from PIL import Image, ImageDraw, ImageFont, ImageOps

VALID_EXTENSIONS = {".jpg", ".jpeg", ".png", ".heic", ".tiff", ".bmp", ".webp"}


def find_font(size):
    """Try a few common system font paths; fall back to PIL's default bitmap font."""
    candidates = [
        "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf",
        "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
        "/System/Library/Fonts/Helvetica.ttc",
        "C:\\Windows\\Fonts\\arialbd.ttf",
        "C:\\Windows\\Fonts\\arial.ttf",
    ]
    for path in candidates:
        if os.path.exists(path):
            try:
                return ImageFont.truetype(path, size)
            except Exception:
                continue
    # Fallback: PIL's built-in font (won't scale well, but never fails)
    return ImageFont.load_default()


def add_text_watermark(img, text, position, opacity, font_size_pct):
    """Overlay semi-transparent text in the given corner. Returns a flattened RGB image."""
    img = img.convert("RGBA")
    width, height = img.size
    long_edge = max(width, height)
    font_size = max(12, int(long_edge * (font_size_pct / 100)))
    font = find_font(font_size)

    overlay = Image.new("RGBA", img.size, (255, 255, 255, 0))
    draw = ImageDraw.Draw(overlay)

    bbox = draw.textbbox((0, 0), text, font=font)
    text_w = bbox[2] - bbox[0]
    text_h = bbox[3] - bbox[1]

    margin = int(long_edge * 0.02)
    positions = {
        "bottom-right": (width - text_w - margin, height - text_h - margin * 2),
        "bottom-left": (margin, height - text_h - margin * 2),
        "top-right": (width - text_w - margin, margin),
        "top-left": (margin, margin),
    }
    x, y = positions.get(position, positions["bottom-right"])

    # subtle drop shadow for legibility on busy backgrounds, then the main text
    shadow_color = (0, 0, 0, min(255, opacity))
    text_color = (255, 255, 255, opacity)
    draw.text((x + 2, y + 2), text, font=font, fill=shadow_color)
    draw.text((x, y), text, font=font, fill=text_color)

    combined = Image.alpha_composite(img, overlay)
    return combined.convert("RGB")


def add_logo_watermark(img, logo_path, position, opacity):
    """Overlay a logo image (PNG with transparency recommended) in the given corner."""
    img = img.convert("RGBA")
    logo = Image.open(logo_path).convert("RGBA")

    width, height = img.size
    long_edge = max(width, height)
    logo_target_w = int(long_edge * 0.15)  # logo ~15% of long edge
    ratio = logo_target_w / logo.width
    logo = logo.resize((logo_target_w, int(logo.height * ratio)), Image.LANCZOS)

    # apply opacity to logo alpha channel
    if opacity < 255:
        alpha = logo.getchannel("A").point(lambda p: int(p * (opacity / 255)))
        logo.putalpha(alpha)

    margin = int(long_edge * 0.02)
    lw, lh = logo.size
    positions = {
        "bottom-right": (width - lw - margin, height - lh - margin),
        "bottom-left": (margin, height - lh - margin),
        "top-right": (width - lw - margin, margin),
        "top-left": (margin, margin),
    }
    x, y = positions.get(position, positions["bottom-right"])

    overlay = Image.new("RGBA", img.size, (255, 255, 255, 0))
    overlay.paste(logo, (x, y), logo)
    combined = Image.alpha_composite(img, overlay)
    return combined.convert("RGB")


def process_image(in_path, out_path, args):
    try:
        img = Image.open(in_path)
        # Respect original orientation (phones store rotation in EXIF), then strip EXIF after
        img = ImageOps.exif_transpose(img)
        img = img.convert("RGB")

        # Resize
        width, height = img.size
        long_edge = max(width, height)
        if long_edge > args.max_size:
            scale = args.max_size / long_edge
            new_size = (int(width * scale), int(height * scale))
            img = img.resize(new_size, Image.LANCZOS)

        # Watermark
        if args.logo:
            img = add_logo_watermark(img, args.logo, args.position, args.opacity)
        else:
            img = add_text_watermark(img, args.watermark_text, args.position, args.opacity, args.font_size_pct)

        # Save — re-saving without passing exif= strips all metadata (GPS, camera, etc.)
        out_path = out_path.with_suffix(".jpg")
        img.save(out_path, "JPEG", quality=args.quality, optimize=True)
        return True, None
    except Exception as e:
        return False, str(e)


def main():
    parser = argparse.ArgumentParser(description="Batch resize, strip metadata, and watermark photos for social posting.")
    parser.add_argument("--input", required=True, help="Folder of original photos")
    parser.add_argument("--output", required=True, help="Folder to save processed photos to")
    parser.add_argument("--watermark-text", default="@uniandrainbowkitty", help="Text watermark (ignored if --logo is used)")
    parser.add_argument("--logo", default=None, help="Path to a logo PNG to use instead of text")
    parser.add_argument("--position", default="bottom-right",
                         choices=["bottom-right", "bottom-left", "top-right", "top-left"])
    parser.add_argument("--max-size", type=int, default=2048, help="Max long-edge dimension in pixels")
    parser.add_argument("--opacity", type=int, default=128, help="Watermark opacity, 0-255")
    parser.add_argument("--font-size-pct", type=float, default=3.0, help="Watermark text size as %% of long edge")
    parser.add_argument("--quality", type=int, default=88, help="JPEG output quality, 1-100")
    args = parser.parse_args()

    in_dir = Path(args.input)
    out_dir = Path(args.output)
    if not in_dir.exists():
        print(f"Input folder not found: {in_dir}")
        sys.exit(1)
    out_dir.mkdir(parents=True, exist_ok=True)

    files = [f for f in in_dir.iterdir() if f.suffix.lower() in VALID_EXTENSIONS]
    if not files:
        print(f"No supported image files found in {in_dir}")
        sys.exit(1)

    print(f"Processing {len(files)} image(s)...")
    ok_count, fail_count = 0, 0
    for f in sorted(files):
        out_path = out_dir / f.name
        success, err = process_image(f, out_path, args)
        if success:
            ok_count += 1
            print(f"  ✓ {f.name}")
        else:
            fail_count += 1
            print(f"  ✗ {f.name} — {err}")

    print(f"\nDone. {ok_count} succeeded, {fail_count} failed.")
    print(f"Output folder: {out_dir.resolve()}")


if __name__ == "__main__":
    main()
