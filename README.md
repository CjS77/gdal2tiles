# gdal2tiles

Generates directory with TMS tiles, KMLs and simple web viewers.

This is a fork of the script available at <http://www.gdal.org/gdal2tiles.html>

Extras: 
 
 * uses EPSG3758 rather than EPSG900913 to be compatible with GDAL on recent Ubuntu/Debian systems
 * allows tiles to be generated in the XYZ format of google maps in addition to TMS format
 * implements parallelisation for faster performance

# SYNOPSIS

    gdal2tiles.py [options ...] input_file [output_dir]
              
# Options:

### --version             
show program's version number and exit

### -h, --help            
show this help message and exit

### -p PROFILE, --profile=PROFILE
Tile cutting profile (mercator,geodetic,raster) . The default is 'mercator' (Google Maps compatible)
 
### -r RESAMPLING, --resampling=RESAMPLING
Resampling method (average,near,bilinear,cubic,cubicspline,lanczos,antialias) - default 'average'

### -s SRS, --s_srs=SRS   
The spatial reference system used for the source input data

### -z ZOOM, --zoom=ZOOM  
Zoom levels to render (format:'2-5' or '10').

### -e, --resume          
Resume mode. Generate only missing files.

### -a NODATA, --srcnodata=NODATA
NODATA transparency value to assign to the input data
 
### -d, --tmscompatible   
**New:** When using the geodetic profile, specifies the base resolution as 0.703125 or 2 tiles at zoom level 0.
 
### -v, --verbose         
Print status messages to stdout

## KML (Google Earth) options:
Options for generated Google Earth SuperOverlay metadata

###   -k, --force-kml     
Generate KML for Google Earth - default for 'geodetic' profile and 'raster' in EPSG:4326. For a dataset with different 
projection use with caution!

###   -n, --no-kml        
Avoid automatic generation of KML files for EPSG:4326 -u URL, --url=URL   URL address where the generated tiles are 
going to be published

## Web viewer options:
Options for generated HTML viewers a la Google Maps

###   -w WEBVIEWER, --webviewer=WEBVIEWER
Web viewer to generate (all,google,openlayers,none) - default 'all'
   
### -t TITLE, --title=TITLE
Title of the map

### -c COPYRIGHT, --copyright=COPYRIGHT
Copyright for the map

### -g GOOGLEKEY, --googlekey=GOOGLEKEY
Google Maps API key from <http://code.google.com/apis/maps/signup.html>
   
### -b BINGKEY, --bingkey=BINGKEY 
Bing Maps API key from <https://www.bingmapsportal.com>              

# DESCRIPTION

This utility generates a directory with small tiles and metadata, following the OSGeo Tile Map Service Specification. 
Simple web pages with viewers based on Google Maps and OpenLayers are generated as well - so anybody can comfortably 
explore your maps on-line and you do not need to install or configure any special software (like MapServer) and the map 
displays very fast in the web browser. You only need to upload the generated directory onto a web server.

GDAL2Tiles also creates the necessary metadata for Google Earth (KML SuperOverlay), in case the supplied map uses 
EPSG:4326 projection.

World files and embedded georeferencing is used during tile generation, but you can publish a picture without proper 
georeferencing too.

This script was tested against GDAL v 1.11.2

# Technical Notes

## Global Map Tiles as defined in Tile Map Service (TMS) Profiles

Functions necessary for generation of global tiles used on the web.
It contains classes implementing coordinate conversions for:

  - GlobalMercator (based on EPSG:3785 = EPSG:3857)
       for Google Maps, Yahoo Maps, Bing Maps compatible tiles
  - GlobalGeodetic (based on EPSG:4326)
       for OpenLayers Base Map and Google Earth compatible tiles

More info at:

http://wiki.osgeo.org/wiki/Tile_Map_Service_Specification
http://wiki.osgeo.org/wiki/WMS_Tiling_Client_Recommendation
http://msdn.microsoft.com/en-us/library/bb259689.aspx
http://code.google.com/apis/maps/documentation/overlays.html#Google_Maps_Coordinates

Created by Klokan Petr Pridal on 2008-07-03.
Google Summer of Code 2008, project GDAL2Tiles for OSGEO.

## TMS Global Mercator Profile

Functions necessary for generation of tiles in Spherical Mercator projection,
EPSG:3857 (EPSG:gOOglE, Google Maps Global Mercator), EPSG:3785, OSGEO:41001, EPSG:3857.

Such tiles are compatible with Google Maps, Bing Maps, Yahoo Maps,
UK Ordnance Survey OpenSpace API, ...
and you can overlay them on top of base maps of those web mapping applications.

Pixel and tile coordinates are in TMS notation (origin [0,0] in bottom-left).
Note: The XYZ notation (option --xyz) uses the XYZ convention where tile (0,0) is on the top-left.

What coordinate conversions do we need for TMS Global Mercator tiles::

     LatLon      <->       Meters      <->     Pixels    <->       Tile     
    
    WGS84 coordinates   Spherical Mercator  Pixels in pyramid  Tiles in pyramid
     lat/lon            XY in metres     XY pixels Z zoom      XYZ from TMS 
    EPSG:4326           EPSG:3857                                         
     .----.              ---------               --                TMS      
    /      \     <->     |       |     <->     /----/    <->      Google    
    \      /             |       |           /--------/          QuadTree   
     -----               ---------         /------------/                   
    KML, public         WebMapService         Web Clients      TileMapService

What is the coordinate extent of Earth in EPSG:3875?

    [-20037508.342789244, -20037508.342789244, 20037508.342789244, 20037508.342789244]

Constant 20037508.342789244 comes from the circumference of the Earth in meters,
which is 40 thousand kilometers, the coordinate origin is in the middle of extent.
In fact you can calculate the constant as: 2 * math.pi * 6378137 / 2.0

    $ echo 180 85 | gdaltransform -s_srs EPSG:4326 -t_srs EPSG:3857

Polar areas with abs(latitude) bigger then 85.05112878 are clipped off.

#### What are zoom level constants (pixels/meter) for pyramid with EPSG:3875?

The whole region is on top of a pyramid (zoom=0) covered by 256x256 pixels tile,
every lower zoom level resolution is always divided by two
initialResolution = 20037508.342789244 * 2 / 256 = 156543.03392804062

#### What is the difference between TMS and Google Maps/QuadTree tile name convention?

The tile raster itself is the same (equal extent, projection, pixel size),
there is just different identification of the same raster tile.
Tiles in TMS are counted from [0,0] in the bottom-left corner, id is XYZ.
Google placed the origin [0,0] to the top-left corner, reference is XYZ.
Microsoft is referencing tiles by a QuadTree name, defined on the website:
http://msdn2.microsoft.com/en-us/library/bb259689.aspx

#### The lat/lon coordinates are using WGS84 datum, yeh?

Yes, all lat/lon we are mentioning should use WGS84 Geodetic Datum.
Well, the web clients like Google Maps are projecting those coordinates by
Spherical Mercator, so in fact lat/lon coordinates on sphere are treated as if
they were on the WGS84 ellipsoid.

*From MSDN documentation*:

>To simplify the calculations, we use the spherical form of projection, not
the ellipsoidal form. Since the projection is used only for map display,
and not for displaying numeric coordinates, we don't need the extra precision
of an ellipsoidal projection. The spherical projection causes approximately
0.33 percent scale distortion in the Y direction, which is not visually noticable.

#### How do I create a raster in EPSG:3875 and convert coordinates with PROJ.4?

You can use standard GIS tools like gdalwarp, cs2cs or gdaltransform.
All of the tools supports -t_srs 'epsg:3857'.

For other GIS programs check the exact definition of the projection:
More info at http://spatialreference.org/ref/user/google-projection/
The same projection is degined as EPSG:3785. WKT definition is in the official
EPSG database.

Proj4 Text:

    +proj=merc +a=6378137 +b=6378137 +lat_ts=0.0 +lon_0=0.0 +x_0=0.0 +y_0=0
    +k=1.0 +units=m +nadgrids=@null +no_defs

Human readable WKT format of EPGS:3857:

     PROJCS["Google Maps Global Mercator",
         GEOGCS["WGS 84",
             DATUM["WGS_1984",
                 SPHEROID["WGS 84",6378137,298.257223563,
                     AUTHORITY["EPSG","7030"]],
                 AUTHORITY["EPSG","6326"]],
             PRIMEM["Greenwich",0],
             UNIT["degree",0.0174532925199433],
             AUTHORITY["EPSG","4326"]],
         PROJECTION["Mercator_1SP"],
         PARAMETER["central_meridian",0],
         PARAMETER["scale_factor",1],
         PARAMETER["false_easting",0],
         PARAMETER["false_northing",0],
         UNIT["metre",1,
                     AUTHORITY["EPSG","9001"]]]
                     
## TMS Global Geodetic Profile

Functions necessary for generation of global tiles in Plate Carre projection,
EPSG:4326, "unprojected profile".

Such tiles are compatible with Google Earth (as any other EPSG:4326 rasters)
and you can overlay the tiles on top of OpenLayers base map.

Pixel and tile coordinates are in TMS notation (origin [0,0] in bottom-left), or XYZ notation (if the `--xyz` option is used)

#### What coordinate conversions do we need for TMS Global Geodetic tiles?

Global Geodetic tiles are using geodetic coordinates (latitude,longitude)
directly as planar coordinates XY (it is also called *Unprojected* or *Plate
Carre*). We need only scaling to pixel pyramid and cutting to tiles.
Pyramid has on top level two tiles, so it is not square but rectangle.
Area [-180,-90,180,90] is scaled to 512x256 pixels.
TMS has coordinate origin (for pixels and tiles) in bottom-left corner.
Rasters are in EPSG:4326 and therefore are compatible with Google Earth.

         LatLon      <->      Pixels      <->     Tiles     
    
     WGS84 coordinates   Pixels in pyramid  Tiles in pyramid
         lat/lon         XY pixels Z zoom      XYZ from TMS 
        EPSG:4326                                           
         .----.                ----                         
        /      \     <->    /--------/    <->      TMS      
        \      /         /--------------/                   
         -----        /--------------------/                
       WMS, KML    Web Clients, Google Earth  TileMapService
                            
                            