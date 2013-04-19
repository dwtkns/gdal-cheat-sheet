Cheat sheet for GDAL/OGR command-line tools

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

__Clip vectors by bounding box__

	ogr2ogr -f "ESRI Shapefile" output.shp input.shp -clipsrc <x_min> <y_min> <x_max> <y_max>

__Clip one vector by another__

	ogr2ogr -clipsrc clipping_polygon.shp output.shp input.shp

__Reproject vector:__

	ogr2ogr output.shp -t_srs "EPSG:4326" input.shp

__Merge vectors:__

	ogr2ogr merged.shp input1.shp
	ogr2ogr -update -append merged.shp input2.shp -nln merged

__Extract from a vector file based on query__

To extract features with STATENAME 'New York','New Hampshire', etc. from states.shp

	ogr2ogr -where 'STATENAME like "New%"' states_subset.shp states.shp

To extract type 'pond' from water.shp

	ogr2ogr -where "type = pond" ponds.shp water.shp

__Subset & filter all shapefiles in a directory__

Assumes that filename and name of layer of interest are the same...  

	ls -1 *.shp | sed 's/.shp//g' | xargs -n1 -I % ogr2ogr %-subset.shp %.shp -sql "SELECT field-one, field-two FROM '%' WHERE field-one='value-of-interest'"

Raster operations
---
__Get raster information__

	gdalinfo input.tif

__List raster drivers__

	gdal_translate --formats

__Convert between raster formats__

	gdal_translate -of "GTiff" input.grd output.tif

__Convert a directory of files to a different raster format__

	ls -1 *.img | sed 's/.img//g' | xargs -n1 -I % gdal_translate -of "GTiff" %.img %.tif

__Reproject raster:__

	gdalwarp -t_srs "EPSG:102003" input.tif output.tif

__Georeference an unprojected image with known bounding coordinates:__

	gdal_translate -of GTiff -a_ullr <top_left_lon> <top_left_lat> <bottom_right_lon> <bottom_right_lat> -a_srs EPSG:4269 input.png output.tif

__Clip raster by bounding box__

	gdalwarp -te <x_min> <y_min> <x_max> <y_max> input.tif clipped_output.tif
	
__Clip raster to SHP / NoData for pixels beyond polygon boundary__

	gdalwarp -dstnodata <nodata_value> -cutline input_polygon.shp input.tif clipped_output.tif

__Merge rasters__

	gdal_merge.py -o merged.tif input1.tif input2.tif

Alternatively,

	gdalwarp input1.tif input2.tif merged.tif
	
Or, to preserve nodata values:

	gdalwarp input1.tif input2.tif merged.tif -srcnodata <nodata_value> -dstnodata <merged_nodata_value>


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

__Resample raster__

	gdalwarp -ts <width> <height> -r cubicspline dem.tif resampled_dem.tif

Entering 0 for either width or height guesses based on current dimensions.

__Burn vector into raster__

	gdal_rasterize -b 1 -i -burn -32678 -l layername input.shp input.tif

__Create contours from DEM__

	gdal_contour -a elev -i 50 input_dem.tif output_contours.shp


Other
---

__Convert KML to CSV (WKT)__  
First list layers in the KML file

	ogrinfo -so input.kml

Convert the desired KML layer to CSV

	ogr2ogr -f CSV output.csv input.kml -sql "select *,OGR_GEOM_WKT from some_kml_layer"

__CSV points to SHP__  
_This section needs retooling_  
Given input.csv

	lon_column,lat_column,value
	-81,32,13
	-81,32,14
	-81,32,15

Make a .dbf table for ogr2ogr to work with from input.csv

	ogr2ogr -f "ESRI Shapefile" input.dbf input.csv

Use a text editor to create a .vrt file in the same directory as input.csv and input.dbf. This file holds the parameters for building a full shapefile based on values in the DBF you just made.

	<OGRVRTDataSource>
	  <OGRVRTLayer name="output_file_name">
	    <SrcDataSource relativeToVRT="1">./</SrcDataSource>
	    <SrcLayer>input</SrcLayer>
	    <GeometryType>wkbPoint</GeometryType>
	    <LayerSRS>WGS84</LayerSRS>
	    <GeometryField encoding="PointFromColumns" x="lon_column" y="lat_column"/>
	  </OGRVRTLayer>
	</OGRVRTDataSource>

Create shapefile based on parameters listed in the .vrt

	mkdir shp
	ogr2ogr -f "ESRI Shapefile" shp/ inputfile.vrt

The VRT file can be modified to give a new output shapefile name, reference a different coordinate system (LayerSRS), or pull coordinates from different columns.

__MODIS operations__

First, download relevant .hdf tiles from the MODIS ftp site: <ftp://ladsftp.nascom.nasa.gov/>; use the [MODIS sinusoidal grid](http://www.geohealth.ou.edu/modis_v5/modis.shtml) for reference.

Create a file containing the names of all .hdf files in the directory

	ls -1 *.hdf > files.txt

List MODIS Subdatasets in a given HDF (conf. the [MODIS products table](https://lpdaac.usgs.gov/products/modis_products_table/))

	gdalinfo longFileName.hdf | grep SUBDATASET

Make TIFs from each file in list; replace 'MOD12Q1:Land_Cover_Type_1' with desired Subdataset name

	mkdir output
	cat files.txt | xargs -I % -n1 gdalwarp -of GTiff 'HDF4_EOS:EOS_GRID:%:MOD12Q1:Land_Cover_Type_1' output/%.tif

Merge all .tifs in output directory into single file

	cd output
	gdal_merge.py -o Merged_Landcover.tif *.tif


Sources
---

<http://live.osgeo.org/en/quickstart/gdal_quickstart.html>

<https://github.com/nvkelso/geo-how-to/wiki/OGR-to-reproject,-modify-Shapefiles>  

<ftp://ftp.remotesensing.org/gdal/presentations/OpenSource_Weds_Andre_CUGOS.pdf>  

<http://developmentseed.org/blog/2009/jul/30/using-open-source-tools-make-elevation-maps-afghanistan-and-pakistan/>  

<http://linfiniti.com/2010/12/a-workflow-for-creating-beautiful-relief-shaded-dems-using-gdal/>  

<http://nautilus.baruch.sc.edu/twiki_dmcc/bin/view/Main/OGR_example>  

<http://www.gdal.org/frmt_hdf4.html>

<http://planetflux.adamwilson.us/2010/06/modis-processing-with-r-gdal-and-nco.html>

<http://trac.osgeo.org/gdal/wiki/FAQRaster>

<http://www.mikejcorey.com/wordpress/2011/02/05/tutorial-create-beautiful-hillshade-maps-from-digital-elevation-models-with-gdal-and-mapnik/>

<http://dirkraffel.com/2011/07/05/best-way-to-merge-color-relief-with-shaded-relief-map/>
