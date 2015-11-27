Cheat sheet for GDAL/OGR command-line geodata tools

Vector operations
---

__Get vector information__

	ogrinfo -so input.shp layer-name

Or, for all layers  

	ogrinfo -al -so input.shp

__Print vector extent__

	ogrinfo input.shp layer-name | grep Extent
	
__List vector drivers__

	ogr2ogr --formats

__Convert between vector formats__

	ogr2ogr -f "GeoJSON" output.json input.shp

__Print count of features with attributes matching a given pattern__

	ogrinfo input.shp layer-name | grep "Search Pattern" | sort | uniq -c

__Read from a zip file__

This assumes that archive.zip is in the current directory. This example just extracts the file, but any ogr2ogr operation should work. It's also possible to write to existing zip files.

	ogr2ogr -f 'GeoJSON' dest.geojson /vsizip/archive.zip/zipped_dir/in.geojson

__Clip vectors by bounding box__

	ogr2ogr -f "ESRI Shapefile" output.shp input.shp -clipsrc <x_min> <y_min> <x_max> <y_max>

__Clip one vector by another__

	ogr2ogr -clipsrc clipping_polygon.shp output.shp input.shp

__Reproject vector:__

	ogr2ogr output.shp -t_srs "EPSG:4326" input.shp

__Add an index to a shapefile__

Add an index on an attribute:

	ogrinfo example.shp -sql "CREATE INDEX ON example USING fieldname"

Add a spatial index:

	ogrinfo example.shp -sql "CREATE SPATIAL INDEX ON example"

__Merge features in a vector file by attribute ("dissolve")__

	ogr2ogr -f "ESRI Shapefile" dissolved.shp input.shp -dialect sqlite -sql "select ST_union(Geometry),common_attribute from input GROUP BY common_attribute"
	
__Merge features ("dissolve") using a buffer to avoid slivers__

	ogr2ogr -f "ESRI Shapefile" dissolved.shp input.shp -dialect sqlite \
	-sql "select ST_union(ST_buffer(Geometry,0.001)),common_attribute from input GROUP BY common_attribute"

__Merge vector files:__

	ogr2ogr merged.shp input1.shp
	ogr2ogr -update -append merged.shp input2.shp -nln merged

__Extract from a vector file based on query__

To extract features with STATENAME 'New York','New Hampshire', etc. from states.shp

	ogr2ogr -where 'STATENAME like "New%"' states_subset.shp states.shp

To extract type 'pond' from water.shp

	ogr2ogr -where "type = pond" ponds.shp water.shp

__Subset & filter all shapefiles in a directory__

Assumes that filename and name of layer of interest are the same...  

	basename -s.shp *.shp | xargs -n1 -I % ogr2ogr %-subset.shp %.shp -sql "SELECT field-one, field-two FROM '%' WHERE field-one='value-of-interest'"

__Extract data from a PostGis database to a GeoJSON file__

	ogr2ogr -f "GeoJSON" file.geojson PG:"host=localhost dbname=database user=user password=password" \
	-sql "SELECT * from table_name"

__Extract data from an ESRI REST API__

Services that use ESRI maps are sometimes powered by a REST server that can provide data in OGR can consume. Finding the correct end point can be tricky and may take some false starts.

	ogr2ogr -f GeoJSON output.geojson "http:/example.com/arcgis/rest/services/SERVICE/LAYER/MapServer/0/query?f=json&returnGeometry=true&etc=..." OGRGeoJSON

__Get the difference between two vector files__

Given two files that both have an id field, this will produce a vector file with the part of `file1.shp` that doesn't intersect with `file2.shp`:

    ogr2ogr diff.shp file1.shp -dialect sqlite \
    -sql "SELECT ST_Difference(a.Geometry, b.Geometry) AS Geometry, a.id \
    FROM file1 a LEFT JOIN 'file2.shp'.file2 b USING (id) WHERE a.Geometry != b.Geometry"

This assumes that `file2.shp` and `file2.shp` are both in the current directory.

__Spatial join:__

A spatial join transfers properties from one vector layer to another based on a [spatial relationship](http://postgis.net/docs/manual-2.0/reference.html#Spatial_Relationships_Measurements) between the features. Fields from the join layer can be [aggregated](https://www.sqlite.org/lang_aggfunc.html) in the output.

Given a set of points (trees.shp) and a set of polygons (parks.shp) in the same directory, create a polygon layer with the geometries from parks.shp and summaries of some columns in trees.shp:

    ogr2ogr -f 'ESRI Shapefile' output.shp parks.shp -dialect sqlite \
    -sql "SELECT p.Geometry, p.id id, SUM(t.field1) field1_sum, AVG(t.field2) field2_avg
    FROM parks p, 'trees.shp'.trees t WHERE ST_Contains(p.Geometry, t.Geometry) GROUP BY p.id"

Note that features that from parks.shp that don't overlap with trees.shp won't be in the new file.

Raster operations
---
__Get raster information__

	gdalinfo input.tif

__List raster drivers__

	gdal_translate --formats
	
__Force creation of world file (requires libgeotiff)__

	listgeo -tfw  mappy.tif
	
__Report PROJ.4 projection info, including bounding box (requires libgeotiff)__

	listgeo -proj4 mappy.tif

__Convert between raster formats__

	gdal_translate -of "GTiff" input.grd output.tif

__Convert 16-bit bands (Int16 or UInt16) to Byte type__  
(Useful for Landsat 8 imagery...)

	gdal_translate -of "GTiff" -co "COMPRESS=LZW" -scale 0 65535 0 255 -ot Byte input_uint16.tif output_byte.tif

You can change '0' and '65535' to your image's actual min/max values to preserve more color variation or to apply the scaling to other band types - find that number with:

	gdalinfo -mm input.tif | grep Min/Max
	
__Convert a directory of raster files of the same format to another raster format__

	basename -s.img *.img | xargs -n1 -I % gdal_translate -of "GTiff" %.img %.tif

__Reproject raster:__

	gdalwarp -t_srs "EPSG:102003" input.tif output.tif
	
Be sure to add _-r bilinear_ if reprojecting elevation data to prevent funky banding artifacts.

__Georeference an unprojected image with known bounding coordinates:__

	gdal_translate -of GTiff -a_ullr <top_left_lon> <top_left_lat> <bottom_right_lon> <bottom_right_lat> \
	-a_srs EPSG:4269 input.png output.tif

__Clip raster by bounding box__

	gdalwarp -te <x_min> <y_min> <x_max> <y_max> input.tif clipped_output.tif
	
__Clip raster to SHP / NoData for pixels beyond polygon boundary__

	gdalwarp -dstnodata <nodata_value> -cutline input_polygon.shp input.tif clipped_output.tif
	
__Crop raster dimensions to vector bounding box__
	
	gdalwarp -cutline cropper.shp -crop_to_cutline input.tif cropped_output.tif

__Merge rasters__

	gdal_merge.py -o merged.tif input1.tif input2.tif

Alternatively,

	gdalwarp input1.tif input2.tif merged.tif
	
Or, to preserve nodata values:

	gdalwarp input1.tif input2.tif merged.tif -srcnodata <nodata_value> -dstnodata <merged_nodata_value>

__Stack grayscale bands into a georeferenced RGB__

Where LC81690372014137LGN00 is a Landsat 8 ID and B4, B3 and B2 correspond to R,G,B bands respectively:

	gdal_merge.py -co "PHOTOMETRIC=RGB" -separate LC81690372014137LGN00_B{4,3,2}.tif -o LC81690372014137LGN00_rgb.tif

__Fix an RGB TIF whose bands don't know they're RGB__

	gdal_merge.py -co "PHOTOMETRIC=RGB" input.tif -o output_rgb.tif

__Export a raster for Google Earth__

	gdal_translate -of KMLSUPEROVERLAY input.tif output.kmz -co FORMAT=JPEG
	
__Raster calculation (map algebra)__

Average two rasters:

	gdal_calc.py -A input1.tif -B input2.tif --outfile=output.tif --calc="(A+B)/2"

Add two rasters:

	gdal_calc.py -A input1.tif -B input2.tif --outfile=output.tif --calc="A+B"

etc.

__Create a hillshade from a DEM__

	gdaldem hillshade -of PNG input.tif hillshade.png

Change light direction:

	gdaldem hillshade -of PNG -az 135 input.tif hillshade_az135.png 

Use correct vertical scaling in meters if input is projected in degrees
	
	gdaldem hillshade -s 111120 -of PNG input_WGS1984.tif hillshade.png

__Apply color ramp to a DEM__  
First, create a color-ramp.txt file:  
_(Height, Red, Green, Blue)_

		0 110 220 110
		900 240 250 160
		1300 230 220 170
		1900 220 220 220
		2500 250 250 250

Then apply those colors to a DEM:

	gdaldem color-relief input.tif color_ramp.txt color-relief.tif

__Create slope-shading from a DEM__  
First, make a slope raster from DEM:

		gdaldem slope input.tif slope.tif 

Second, create a color-slope.txt file:  
_(Slope angle, Red, Green, Blue)_

	0 255 255 255
	90 0 0 0  

Finally, color the slope raster based on angles in color-slope.txt:  

	gdaldem color-relief slope.tif color-slope.txt slopeshade.tif

__Resample (resize) raster__

	gdalwarp -ts <width> <height> -r cubic dem.tif resampled_dem.tif

Entering 0 for either width or height guesses based on current dimensions.

Alternatively,
	
	gdal_translate -outsize 10% 10% -r cubic dem.tif resampled_dem.tif

For both of these, `-r cubic` specifies cubic interpolation: when resampling continuous data (like a DEM), the default nearest neighbor interpolation can result in "stair step" artifacts.

__Burn vector into raster__

	gdal_rasterize -b 1 -i -burn -32678 -l layername input.shp input.tif

__Extract polygons from raster__

	gdal_polygonize.py input.tif -f "GeoJSON" output.json

__Create contours from DEM__

	gdal_contour -a elev -i 50 input_dem.tif output_contours.shp

__Get values for a specific location in a raster__

	gdallocationinfo -xml -wgs84 input.tif <lon> <lat>  

__Convert GRIB band to .tif__
	
Assumes data for entire globe in WGS84. Be sure to specify band, or you may end up with a nonsense RGB image composed of the first three.

	gdal_translate input.grib -a_ullr -180 -90 180 90 -a_srs EPSG:4326 -b 1 output_band1.tif
	

Other
---
__Convert KML points to CSV (simple)__

	ogr2ogr -f CSV output.csv input.kmz -lco GEOMETRY=AS_XY

__Convert KML to CSV (WKT)__  
First list layers in the KML file

	ogrinfo -so input.kml

Convert the desired KML layer to CSV

	ogr2ogr -f CSV output.csv input.kml -sql "select *,OGR_GEOM_WKT from some_kml_layer"

__CSV points to SHP__  

Given `input.csv`:

	lon,lat,value
	-81,31,13
	-80,32,14
	-81,33,15

Create a shapefile, using Spatialite functions to generate the point:

	ogr2ogr output.shp input.csv -dialect sqlite \
	-sql "SELECT MakePoint(CAST(lon as REAL), CAST(lat as REAL), 4326) Geometry, * FROM input"

Note the 4326, which refers to a spatial reference (in this case [`EPSG:4326`](http://epsg.io/4326)). Use the correct code for your data.

__MODIS operations__

First, download relevant .hdf tiles from the MODIS ftp site: <ftp://ladsftp.nascom.nasa.gov/>; use the [MODIS sinusoidal grid](http://www.geohealth.ou.edu/modis_v5/modis.shtml) for reference.

List MODIS Subdatasets in a given HDF (conf. the [MODIS products table](https://lpdaac.usgs.gov/products/modis_products_table/))

	gdalinfo longFileName.hdf | grep SUBDATASET

Make TIFs from each file in list; replace 'MOD12Q1:Land_Cover_Type_1' with desired Subdataset name

	mkdir output
	find . '*.hdf' -exec gdalwarp -of GTiff 'HDF4_EOS:EOS_GRID:"{}":MOD12Q1:Land_Cover_Type_1' output/{}.tif \;

Merge all .tifs in output directory into single file

	gdal_merge.py -o output/Merged_Landcover.tif output/*.tif

__BASH functions__  
_Size Functions_  
This size function echos the pixel dimensions of a given file in the format expected by gdalwarp.

	function gdal_size() {
		SIZE=$(gdalinfo $1 |\
			grep 'Size is ' |\
			cut -d\   -f3-4 |\
			sed 's/,//g')
		echo -n "$SIZE"
	}

This can be used to easily resample one raster to the dimensions of another:

	gdalwarp -ts $(gdal_size bigraster.tif) -r cubicspline smallraster.tif resampled_smallraster.tif

_Extent Functions_  
These extent functions echo the extent of the given file in the order/format expected by gdal_translate -projwin.
(Originally from [Linfiniti](http://linfiniti.com/2009/09/clipping-rasters-with-gdal-using-polygons/)).

	function gdal_extent() {
		if [ -z "$1" ]; then 
			echo "Missing arguments. Syntax:"
			echo "  gdal_extent <input_raster>"
	    	return
		fi
		EXTENT=$(gdalinfo $1 |\
			grep "Upper Left\|Lower Right" |\
			sed "s/Upper Left  //g;s/Lower Right //g;s/).*//g" |\
			tr "\n" " " |\
			sed 's/ *$//g' |\
			tr -d "[(,]")
		echo -n "$EXTENT"
	}

	function ogr_extent() {
		if [ -z "$1" ]; then 
			echo "Missing arguments. Syntax:"
			echo "  ogr_extent <input_vector>"
	    	return
		fi
		EXTENT=$(ogrinfo -al -so $1 |\
			grep Extent |\
			sed 's/Extent: //g' |\
			sed 's/(//g' |\
			sed 's/)//g' |\
			sed 's/ - /, /g')
		EXTENT=`echo $EXTENT | awk -F ',' '{print $1 " " $4 " " $3 " " $2}'`
		echo -n "$EXTENT"
	}

	function ogr_layer_extent() {
		if [ -z "$2" ]; then 
			echo "Missing arguments. Syntax:"
			echo "  ogr_extent <input_vector> <layer_name>"
	    	return
		fi
		EXTENT=$(ogrinfo -so $1 $2 |\
			grep Extent |\
			sed 's/Extent: //g' |\
			sed 's/(//g' |\
			sed 's/)//g' |\
			sed 's/ - /, /g')
		EXTENT=`echo $EXTENT | awk -F ',' '{print $1 " " $4 " " $3 " " $2}'`
		echo -n "$EXTENT"
	}

Extents can be passed directly into a gdal_translate command like so:

	gdal_translate -projwin $(ogr_extent boundingbox.shp) input.tif clipped_output.tif
	
or
	
	gdal_translate -projwin $(gdal_extent target_crop.tif) input.tif clipped_output.tif

This can be a useful way to quickly crop one raster to the same extent as another. Add these to your ~/.bash_profile file for easy terminal access.


Sources
---

<http://live.osgeo.org/en/quickstart/gdal_quickstart.html>

<https://github.com/nvkelso/geo-how-to/wiki/OGR-to-reproject,-modify-Shapefiles>  

<ftp://ftp.remotesensing.org/gdal/presentations/OpenSource_Weds_Andre_CUGOS.pdf>  

<http://developmentseed.org/blog/2009/jul/30/using-open-source-tools-make-elevation-maps-afghanistan-and-pakistan/>  

<http://linfiniti.com/2010/12/a-workflow-for-creating-beautiful-relief-shaded-dems-using-gdal/>  

<http://linfiniti.com/2009/09/clipping-rasters-with-gdal-using-polygons/>  

<http://nautilus.baruch.sc.edu/twiki_dmcc/bin/view/Main/OGR_example>  

<http://www.gdal.org/frmt_hdf4.html>

<http://planetflux.adamwilson.us/2010/06/modis-processing-with-r-gdal-and-nco.html>

<http://trac.osgeo.org/gdal/wiki/FAQRaster>

<http://www.mikejcorey.com/wordpress/2011/02/05/tutorial-create-beautiful-hillshade-maps-from-digital-elevation-models-with-gdal-and-mapnik/>

<http://dirkraffel.com/2011/07/05/best-way-to-merge-color-relief-with-shaded-relief-map/>

<http://gfoss.blogspot.com/2008/06/gdal-raster-data-tips-and-tricks.html>

<http://osgeo-org.1560.x6.nabble.com/gdal-dev-Dissolve-shapefile-using-GDAL-OGR-td5036930.html>

<https://www.mapbox.com/tilemill/docs/guides/terrain-data/>

<https://gist.github.com/ashaw/0862ec044c45b9aa3c76>

<https://github.com/gina-alaska/dans-gdal-scripts>

