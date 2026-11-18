# WordPress Media Processing Diagnostics

A practical troubleshooting reference for diagnosing and fixing WordPress media upload and image processing problems.

This repository covers common issues involving image uploads, thumbnail generation, image editors, GD, Imagick, MIME types, attachment metadata, and server resource limits.

## Table of Contents

- [Overview](#overview)
- [Common Media Problems](#common-media-problems)
- [1. Check Server Image Support](#1-check-server-image-support)
- [2. Check PHP Upload Configuration](#2-check-php-upload-configuration)
- [3. Diagnose Upload Failures](#3-diagnose-upload-failures)
- [4. Check Image Editor Support](#4-check-image-editor-support)
- [5. Diagnose Thumbnail Generation](#5-diagnose-thumbnail-generation)
- [6. Check Registered Image Sizes](#6-check-registered-image-sizes)
- [7. Check Attachment Metadata](#7-check-attachment-metadata)
- [8. Diagnose MIME Type Issues](#8-diagnose-mime-type-issues)
- [9. Fix Image Orientation](#9-fix-image-orientation)
- [10. Regenerate Missing Thumbnails](#10-regenerate-missing-thumbnails)
- [11. Check WebP Processing](#11-check-webp-processing)
- [12. Check Large Image Problems](#12-check-large-image-problems)
- [Useful WP-CLI Commands](#useful-wp-cli-commands)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Best Practices](#best-practices)
- [Contributing](#contributing)
- [License](#license)

## Overview

WordPress uses its media handling system to upload files, create attachment records, generate image sizes, and process image metadata.

Media processing problems can occur because of:

- Missing PHP image extensions
- Incorrect server upload limits
- Unsupported image formats
- Insufficient memory
- Image editor failures
- Incorrect file permissions
- Failed thumbnail generation
- Missing attachment metadata
- Incompatible image optimization tools

This guide provides practical checks to help identify these problems.

## Common Media Problems

| Problem | Possible Cause |
|---|---|
| Image upload fails | Upload limits or permissions |
| HTTP error during upload | Server or image processing failure |
| Thumbnails are missing | Image editor or memory issue |
| Images appear rotated | EXIF orientation handling |
| WebP upload is rejected | MIME type or format support |
| Image sizes are missing | Registration or generation issue |
| Media library image is broken | Missing file or metadata |
| Large images fail to process | Memory or execution limits |

## 1. Check Server Image Support

WordPress relies on PHP image processing capabilities.

Check whether GD and Imagick are available:

```bash
php -m
```

Look for:

```text
gd
imagick
```

You can also check PHP configuration:

```bash
php -i | grep -i -E "gd|imagick"
```

On Windows:

```powershell
php -m | findstr /i "gd imagick"
```

Check the extensions loaded by the PHP version used by the web server, not only the command-line PHP installation.

## 2. Check PHP Upload Configuration

Several PHP settings affect media uploads.

Important configuration values:

```ini
file_uploads = On
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 120
max_input_time = 120
```

These are example values, not universal requirements.

Check the active configuration:

```bash
php -i | grep -E "upload_max_filesize|post_max_size|memory_limit|max_execution_time"
```

On Windows:

```powershell
php -i | findstr /i "upload_max_filesize post_max_size memory_limit max_execution_time"
```

Remember:

- `post_max_size` should be larger than or equal to `upload_max_filesize`.
- The web server or hosting provider may impose additional limits.
- PHP CLI and PHP-FPM or Apache can load different configuration files.

## 3. Diagnose Upload Failures

When an image cannot be uploaded, check the following:

1. Confirm that the file format is supported.
2. Check the file size.
3. Check the PHP upload limits.
4. Verify that the uploads directory is writable.
5. Check available disk space.
6. Review PHP and web server error logs.
7. Temporarily test without image optimization or upload-related plugins.

Check the WordPress uploads directory:

```bash
wp eval 'echo wp_upload_dir()["basedir"] . PHP_EOL;'
```

Check the directory permissions on Linux:

```bash
ls -ld wp-content/uploads
```

Avoid applying broad `777` permissions. Use the ownership and permissions recommended by the hosting provider.

## 4. Check Image Editor Support

WordPress can use available image processing libraries such as Imagick and GD.

Check which image editor WordPress selects:

```bash
wp eval '
$editor = wp_get_image_editor(
    ABSPATH . "wp-includes/images/media/default.png"
);

if ( is_wp_error( $editor ) ) {
    echo $editor->get_error_message() . PHP_EOL;
} else {
    echo get_class( $editor ) . PHP_EOL;
}
'
```

Possible results include:

```text
WP_Image_Editor_Imagick
WP_Image_Editor_GD
```

If the editor cannot be initialized, investigate:

- Missing PHP extensions
- Unsupported image formats
- Broken image files
- Image library configuration
- Server memory restrictions

## 5. Diagnose Thumbnail Generation

WordPress creates additional image sizes based on registered settings.

When thumbnails are missing:

1. Check whether the original image exists.
2. Confirm that the image can be opened by the image editor.
3. Check registered image sizes.
4. Review memory and execution limits.
5. Check for errors in PHP logs.
6. Test thumbnail regeneration on a small number of attachments.

Do not repeatedly regenerate every image before identifying the underlying issue.

## 6. Check Registered Image Sizes

List the image sizes registered with WordPress:

```bash
wp media image-size
```

Example:

```text
name          width   height   crop
thumbnail     150     150      hard
medium        300     300      soft
medium_large  768     0        soft
large         1024    1024     soft
full
```

To register a custom image size, use:

```php
add_action( 'after_setup_theme', function () {
    add_image_size( 'custom-card', 600, 400, true );
} );
```

The custom size applies to newly generated image metadata and future uploads. Existing attachments may require thumbnail regeneration.

## 7. Check Attachment Metadata

WordPress stores information about uploaded media in attachment metadata.

Inspect an attachment:

```bash
wp eval '
$id = 123;

$file = get_attached_file( $id );
$metadata = wp_get_attachment_metadata( $id );

echo "File: " . $file . PHP_EOL;
echo "Metadata:" . PHP_EOL;

print_r( $metadata );
'
```

Replace `123` with a valid attachment ID.

Check for:

- Missing original file
- Empty or incomplete metadata
- Missing generated image sizes
- Incorrect dimensions
- Incorrect metadata file paths

If the original file is missing, regenerating thumbnails will not restore it.

## 8. Diagnose MIME Type Issues

WordPress validates uploaded file types.

Check the detected MIME type:

```bash
wp eval '
$file = "/path/to/image.webp";

$result = wp_check_filetype( $file );

print_r( $result );
'
```

Replace the example path with an actual file path.

To inspect allowed upload MIME types:

```bash
wp eval 'print_r( get_allowed_mime_types() );'
```

Only add custom MIME types when the format is trusted and supported by the server.

Example:

```php
add_filter( 'upload_mimes', function ( $mimes ) {
    $mimes['webp'] = 'image/webp';

    return $mimes;
} );
```

Allowing a MIME type does not guarantee that the server can process the file.

## 9. Fix Image Orientation

Some images may appear rotated because of EXIF orientation metadata.

WP-CLI provides an orientation correction command:

```bash
wp media fix-orientation 123
```

Replace `123` with the attachment ID.

To check the command options:

```bash
wp help media fix-orientation
```

Always test orientation corrections on a small number of images first.

## 10. Regenerate Missing Thumbnails

WP-CLI can regenerate image sizes for existing attachments.

Regenerate thumbnails for one attachment:

```bash
wp media regenerate 123
```

Regenerate only missing image sizes:

```bash
wp media regenerate --only-missing
```

Regenerate a specific image size:

```bash
wp media regenerate --image_size=large
```

Regenerate thumbnails for all attachments:

```bash
wp media regenerate --yes
```

Use the full regeneration command carefully on large media libraries.

Before running bulk operations:

- Back up the uploads directory.
- Confirm that the original images are available.
- Check available disk space.
- Test with a few attachment IDs.
- Run the command during a low-traffic period.

## 11. Check WebP Processing

WebP upload and processing require appropriate support from WordPress, PHP, and the image editor.

Check whether WordPress recognizes WebP:

```bash
wp eval '
$file = wp_check_filetype( "sample.webp" );
print_r( $file );
'
```

Check the installed PHP extensions:

```bash
php -m
```

If WebP processing fails, investigate:

- Whether the active image editor supports WebP
- Whether the PHP image library was built with WebP support
- Whether the uploaded file is valid
- Whether the server has enough memory
- Whether an optimization plugin is interfering

Allowing WebP uploads and successfully generating WebP images are separate capabilities.

## 12. Check Large Image Problems

Large images can require significant memory during processing.

A high-resolution image may use much more memory during decoding than its original file size.

Check:

```bash
php -i | grep -E "memory_limit|max_execution_time"
```

Investigate:

- PHP memory limits
- Execution time limits
- Available server memory
- Image dimensions
- Image editor support
- Hosting-level resource restrictions

Do not increase resource limits blindly. Identify the failing operation and use values appropriate for the server.

## Useful WP-CLI Commands

| Command | Purpose |
|---|---|
| `wp media image-size` | List registered image sizes |
| `wp media regenerate --only-missing` | Generate missing image sizes |
| `wp media regenerate 123` | Regenerate sizes for one attachment |
| `wp media fix-orientation 123` | Correct image orientation |
| `wp media import image.jpg` | Import an image as an attachment |
| `wp media prune --yes` | Remove generated thumbnails |
| `wp help media` | Display media command help |

Use `wp media prune` with care. It removes generated thumbnail files and should not be treated as a general cleanup command without reviewing its effects.

## Troubleshooting Checklist

Use this checklist before making server or WordPress configuration changes.

- [ ] Confirm the original image exists.
- [ ] Verify the file format.
- [ ] Check PHP upload limits.
- [ ] Check PHP memory limits.
- [ ] Verify GD or Imagick availability.
- [ ] Confirm the uploads directory is writable.
- [ ] Check available disk space.
- [ ] Review PHP and server error logs.
- [ ] Verify registered image sizes.
- [ ] Inspect attachment metadata.
- [ ] Test thumbnail generation.
- [ ] Check WebP support if relevant.
- [ ] Test without image optimization plugins.
- [ ] Back up files before bulk regeneration.

## Best Practices

- Keep PHP image processing extensions updated.
- Use supported image formats.
- Avoid unnecessary custom image sizes.
- Keep original uploaded images backed up.
- Test bulk operations before running them on large libraries.
- Avoid unrestricted MIME type changes.
- Monitor server memory and disk usage.
- Keep image optimization tools compatible with the active image editor.
- Use WP-CLI for controlled media maintenance.

## Contributing

Contributions are welcome.

You can help by:

- Reporting media processing issues.
- Adding troubleshooting examples.
- Improving diagnostic commands.
- Documenting hosting-specific problems.
- Correcting outdated instructions.

Please provide WordPress, PHP, and image library versions when reporting environment-specific issues.

## License

This project is available under the MIT License.

You are free to use, modify, and distribute the material with appropriate attribution.

---

**WordPress Media Processing Diagnostics**  
A practical reference for troubleshooting WordPress media and image processing problems.
