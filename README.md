Cheat sheet for GDAL/OGR command-line tools

Vector operations
---

__Get vector information__

	ogrinfo -so input.shp

__Print vector extent__

	ogrinfo input.shp layername | grep Extent

__List vector drivers__

	ogr2ogr --formats

__Convert between vector formats__

	ogr2ogr -f "GeoJSON" output.json input.shp

__Clip vectors by bounding box__

	ogr2ogr -f "ESRI Shapefile" output.shp input.shp -clipsrc x_min y_min x_max y_max

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


Raster operations
---
__Get raster information__

	gdalinfo input.tif

__List raster drivers__

	gdal_translate --formats

__Convert between raster formats__

	gdal_translate -of "GeoTIFF" input.grd output.tif

__Reproject raster:__

	gdalwarp -t_srs "EPSG:102003" input.tif output.tif

__Merge rasters__

	gdal_merge.py -o merged.tif input1.tif input2.tif

Alternatively, to preserve nodata values:

	gdalwarp input1.tif input2.tif merged.tif -srcnodata <NoData value> -dstnodata <merged NoData value>


__Create a hillshade from a DEM__

	gdaldem hillshade -of PNG input.tif hillshade.png

Change light direction:

	gdaldem hillshade -of PNG -az 135 input.tif hillshade_az135.png 


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
Given input.csv

	lon,lat,value
	-81,32,13
	-81,32,14
	-81,32,15

Make a .dbf table from input.csv

	ogr2ogr -f "ESRI Shapefile" output_dir/output.dbf input.csv

Use a text editor to create a .vrt file in the same directory

	<OGRVRTDataSource>
	  <OGRVRTLayer name="output">
	    <SrcDataSource relativeToVRT="1">/output_dir</SrcDataSource>
	    <SrcLayer>input</SrcLayer>
	    <GeometryType>wkbPoint</GeometryType>
	    <LayerSRS>WGS84</LayerSRS>
	    <GeometryField encoding="PointFromColumns" x="lon" y="lat"/>
	  </OGRVRTLayer>
	</OGRVRTDataSource>

Create shapefile based on parameters listed in the .vrt

	ogr2ogr -f "ESRI Shapefile" /output_dir parameters.vrt

Sources
---

<http://live.osgeo.org/en/quickstart/gdal_quickstart.html>

<https://github.com/nvkelso/geo-how-to/wiki/OGR-to-reproject,-modify-Shapefiles>  

<ftp://ftp.remotesensing.org/gdal/presentations/OpenSource_Weds_Andre_CUGOS.pdf>  

<http://developmentseed.org/blog/2009/jul/30/using-open-source-tools-make-elevation-maps-afghanistan-and-pakistan/>  

<http://linfiniti.com/2010/12/a-workflow-for-creating-beautiful-relief-shaded-dems-using-gdal/>  

<http://nautilus.baruch.sc.edu/twiki_dmcc/bin/view/Main/OGR_example>  
