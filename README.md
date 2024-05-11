# Timelapse 360° photos processing
The purpose of this guide is to document the steps, which are necessary to mitigate these shortcomings:

## Problem #1: GPS position displacement caused by GoPro MAX in Timelapse mode
It seems, that processing of the Timelapse photo by GoPro MAX is taking quite a long time and unfortunately it is recording the GPS position **after** the photo is saved, not **before** (as expected).

**Example:** Photo is taken in the middle of a pedestrian crossing (see the preview of the photo in the bottom-right corner), but the GPS position (in the EXIF data) points to a distant location. The red dot indicates the correct GPS position of this photo.

![GPS position displacement of GoPro MAX](/images/josm_photo_displacement.jpg)

## Problem #2: Inability of Mapillary command line tools to remove duplicates of 360° photos
Despite the fact, that the Mapillary tools should remove duplicates based on the distance between the individual photos only, i.e. with ignoring the photo "direction", it fails to do so.

**Command used:**
```
mapillary_tools process /data/photos/360pano/_upload/ --desc_path /tmp/mapillary_upload.json --duplicate_distance 3 --duplicate_angle 360
```

**Screenshot of multiple photos on the same position:**

![Screenshot of duplicates in Mapillary](/images/mapillary_duplicates.jpg)


## How to fix?
**1) Create a GPX track from the photos:**
  - Install [ExifTool](https://exiftool.org/), for example in Debian: `sudo apt install libimage-exiftool-perl`.
  - Download the format [file](https://github.com/exiftool/exiftool/blob/master/fmt_files/gpx.fmt) for generating a GPX track log.
  - Execute: `exiftool -if '$gpsdatetime' -fileOrder gpsdatetime -p gpx.fmt /data/photos/360pano/_incoming/101GOPRO/ > /data/photos/360pano/_incoming/101gopro.gpx`


**2) Adjust the photos position in the JOSM editor:**
  - Start [JOSM](https://josm.openstreetmap.de/) editor, and install the plugin [photo_geotagging](https://wiki.openstreetmap.org/wiki/JOSM/Plugins/Photo_Geotagging).
  - Open the photos, together with the GPX created in the previous step:

    ![JOSM open dialog](/images/josm_open_photos_with_gpx.png)
  - From the menu **Imagery** select your favourite aerial map. In case, that such aerial map is misaligned (e.g. Bing), please align it before continuing!
  - Choose a photo, that was taken at the position, which is clearly identifiable on the aerial map, e.g. road junction, pedestrian crossing, etc.
  - In the **Layers** panel (top-right corner), select **Geotagged Images** and from its context menu choose **Correlate to GPX**:

    ![JOSM layers](/images/josm_layers.png)
  - Fill in the dialog as follows:
    1) _GPX track_: the currently opened GPX track should be preselected, if not, change it accordingly.
    2) _Timezone_: if you are getting the error "No images could be matched!" or the photo markers disappear from the map, then the timezone is not set correctly. Also pay attention to the summer/winter time (e.g. in the central Europe it is +1:00 during the winter, but +2:00 in the summer).
    3) _Offset_: Find a value, which is best suitable for you. The photo markers are automatically moved on the map, even without closing this dialog, but you have to **leave** this input box, e.g. click to the Timezone input box.
    4) _Override position_: the first checkbox has to be selected, otherwise the changes above will not take affect.
    5) _Set image direction_: should be selected, so there is only one Mapillary **process** command necessary to upload the photos. If not checked, then you will need two Mapillary **process** commands to fix the issue with removing the duplicates before upload (not described here).

    ![JOSM correlate images with GPX](/images/josm_correlate_images_with_gpx.png)
  - Write coordinates back to the photos:

    ![JOSM write coordinates](/images/josm_write_coordinates.png)
  - If you want to verify, that everything went well, then close the layers "Geotagged images" and GPX, and open the directory with your photos again. This time the photo markers will be placed correctly on the map.

**3) Split the photo collection** (from the "_incoming" directory) into smaller groups (each as one subdirectory in the "_upload" directory), representing road branches to avoid duplication of photos on the main roads.

**4) Upload the photos using the Mapillary command line tools:**
  - Process the photos:
    ```
    mapillary_tools process /data/photos/360pano/_upload/ --desc_path /tmp/mapillary_fixed.json --duplicate_distance 3 --duplicate_angle 360 --interpolate_directions
    ```
  - Now you can check in the JSON file, that duplicates are marked as **MapillaryDuplicationError**, e.g.:
    ```
    {
        "error":{
        "type":"MapillaryDuplicationError",
        "message":"Duplicate of its previous image in terms of distance <= 3.0 and angle <= 360.0",
        "vars":{
            "desc":{
            "filename":"/data/photos/360pano/_upload/120240407/GSAB1020.JPG",
            "md5sum":"51aae086bea814be3406936c44b9e359",
            "filetype":"image",
            "MAPLatitude":49.2230333,
            "MAPLongitude":18.7739076,
            "MAPCaptureTime":"2024_04_07_13_24_27_000",
            "MAPAltitude":383.852,
            "MAPCompassHeading":{
                "TrueHeading":109.448,
                "MagneticHeading":109.448
            },
            "MAPDeviceMake":"GoPro",
            "MAPDeviceModel":"GoPro Max",
            "MAPOrientation":1
            },
            "distance":1.9766285054741997,
            "angle_diff":0.7292049758438708
        }
        },
        "filename":"/data/photos/360pano/_upload/20240407/GSAB1020.JPG",
        "filetype":"image"
    },
    ```
  - Upload, if the result is as expected:
    ```
    mapillary_tools upload /data/photos/360pano/_upload/ --desc_path /tmp/mapillary_fixed.json  --user_name "your_username" --organization_key "your_organization_id"
    ```
