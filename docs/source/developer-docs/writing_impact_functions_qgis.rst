.. _writing_impact_functions_qgis:

Writing Impact Functions Using QGIS API
========================================

This document explains the purpose of impact functions and shows how to write them. Starting from InaSAFE 2.0, impact functions can be written using the QGIS API. It is still possible to use native InaSAFE API to write impact functions but for someone with a GIS background and/or familiar with the QGIS API, this could be an easier way to accomplish the same task.
The following sections illustrates the major concepts that differ from writing impact functions using native inasafe layers (:ref:`writing_impact_functions`). 
 

.. note:: This section is still work in progress

What is the major difference from the previous way?
--------------------------------------------------

The major concepts such as dealing with one hazard and one exposure data are still the same. The major difference is that instead of dealing with safe native layers, the input data in these impact functions is a QGIS layer of type QgsVectorLayer and QgsRasterLayer. Impact analysis processing is then done using mainly QGIS API. It is important to review and have a good understanding of the QGIS API documentation. Here are some useful links:

* http://qgis.org/api/
* http://www.qgis.org/en/docs/pyqgis_developer_cookbook/
* http://www.qgis.org/en/docs/pyqgis_developer_cookbook/vector.html
* http://www.qgis.org/en/docs/pyqgis_developer_cookbook/raster.html


Creating impact functions
-------------------------

Creation of impact functions follow the same structure described in :ref:`writing_impact_functions`. 


Import required modules
.......................

Module import concept is similar to what is described in :ref:`writing_impact_functions`. As a minimum, one must import functionality specific to the impact function framework, but depending on the usage other standard Python modules may be imported here. A minimal import section contains:
::

  from safe.impact_functions.core import (FunctionProvider,
                                            get_hazard_layer,
                                            get_exposure_layer,
                                            get_question,
                                            get_function_title)

  from safe.common.tables import Table, TableRow

The main difference to note is that layers returned from the utility methods are now of a QGIS type:

`get_hazard_layer <../api-docs/safe/impact_functions/core.html#safe.impact_functions.core.get_hazard_layer>`_
    Helper function to extract hazard layer from input. For QGIS  type impact functions, the return object `QgisWrapper <../api-docs/safe_qgis/utilities/qgis_layer_wrapper.html`_ type.

`get_exposure_layer <../api-docs/safe/impact_functions/core.html#safe.impact_functions.core.get_exposure_layer>`_ 
    Helper function to extract exposure layer from input. For QGIS  type impact functions, the return object `QgisWrapper <../api-docs/safe_qgis/utilities/qgis_layer_wrapper.html`_ type.

Here is how they should be used:

::
  
    QgisWrapperHazardLayer = get_hazard_layer(layers)
    QgisNativeHazardLayer = QgisWrapperHazardLayer.get_layer()
 

Beside the modules described above, a series of inasafe utility modules dealing with QGIS vector or raster layers need to be imported. It is also possible that some QGIS modules dealing with geometry and coordinates system might be needed in the impact functions. Here are some of the modules that you might need:

::

   from safe.common.qgis_vector_tools import split_by_polygon, clip_by_polygon
   from safe.common.qgis_raster_tools import polygonize, clip_raster

   from qgis.core import (
       QgsRectangle,
       QgsGeometry,
       QgsCoordinateReferenceSystem,
       QgsCoordinateTransform
    )


For details on QGIS related imports and the api they provide, please refer to http://qgis.org/api/


Mandatory methods
.................

The following two methods must be defined inside your impact function class. The return value (qgis2.0) of  get_function_type identify your impact function as being able to work with QGIS input layers. The set_extent is used to set the extent of the area of interest.

::

    def get_function_type(self):
        """Get type of the impact function.

        :returns:   'qgis2.0'
        """
        return 'qgis2.0'

    def set_extent(self, extent):
        """
        Set up the extent of area of interest ([xmin, ymin, xmax, ymax]).

        Mandatory method.

        :param extent: Extent for the analysis.
        """
        self.extent = extent


Some of utility functions available to the impact functions
...........................................................

`clip_by_polygon <../api-docs/safe/common/qgis_vector_tools.html#safe.common.qgis_vector_tools.clip_by_polygon>`_
    Helper function to clip a vector layer. All features inside the polygons are returned. Features that intersect the polygon are clipped and the part the fits in the polygon is returned. It can be used to clip your data to the extents of the current view.


`split_by_polygon <../api-docs/safe/common/qgis_vector_tools.html#safe.common.qgis_vector_tools.split_by_polygon>`_
    Helper function to split vector elements that lie inside a polygon. For example if a line feature crosses a polygon, It is split between the part that lies inside and the part that lies outside the polygon. A marker attribute can be added to both features with a predefined value for features that are inside the polygon.


`clip_raster <../api-docs/safe/common/qgis_raster_tools.html#safe.common.qgis_raster_tools.clip_raster>`_
    Helper function to clip a raster layer based on given extents. It can be used to clip your data to the extents of the current view.

`polygonize <../api-docs/safe/common/qgis_raster_tools.html#safe.common.qgis_raster_tools.polygonize>`_
    Helper function that transforms raster pixels into a vector polygons. Depending on the settings (such as threshold values), only some pixels that respect the setting are considered. The general flow is that each valid pixel is transformed into a polygon and the resulting polygons are merged into one single polygon. 


.. _raster_line:

Example of an Impact function using a raster hazard and line vector exposure
----------------------------------------------------------------------------

The example below comes from flood on roads impact function. The main ideas in this functions are:

* clip the raster to a defined extent
* 'polygonize' the raster, creating one polygon
* split the road layer into features that either are inside or outside the polygon
* calculate the length of impacted roads

Import section
..............

This section identifies functionality that is needed for the impact function in question. One must import functionality specific to the impact function framework, but in many cases, we need to import QGIS core utility classes as well as InaSAFE utility classes. Here is a simple example:


::

  from qgis.core import (
        QgsRectangle,
        QgsFeatureRequest,
        QgsGeometry,
        QgsCoordinateReferenceSystem,
        QgsCoordinateTransform
    )
  
   from safe.common.qgis_raster_tools import polygonize, clip_raster
   from safe.common.qgis_vector_tools import split_by_polygon, clip_by_polygon

    

Impact function class
.....................

Impact function class is described in :ref:`writing_impact_functions` 


Impact function algorithm
.........................

The actual calculation of the impact function is specified as a method call
called ``run``. This is similar to :ref:`writing_impact_functions`. Here are some of the essentials of this function:
::

    def run(self, layers):
        """Impact function for polygon flood on roads

        Input
	  layers: List of layers expected to contain
              H: Polygon layer of inundation areas
              E: Vector layer of roads

        Identifies roads impacted by the flood

        Return
          A layer where flooded and non flooded roads are identified.
	  Calculates the total length of impacted roads.
        """

        # Identify hazard and exposure layers
	H = get_hazard_layer(layers)    # Flood
        E = get_exposure_layer(layers)  # Roads

	#Get QGIS vector and raster layers.
	H = H.get_layer()
        E = E.get_layer()
        
        .....

The typical way to start is by clipping the raster data to the AOI (area of interest) extent defined by the user. Steps are:

* find the raster extent
* find the intersection extent between the raster and the AOI
* use inasafe utility methods to clip the raster 

Here are some code that can be used:
	
::

   #get the raster extent
   raster_extent = H.dataProvider().extent()    
   #find intersection between self.extent and raster_extent
   ...
   #clip the raster
   small_raster = clip_raster(H, width, height, QgsRectangle(*clip_extent))


The next step is to "polygonize" the raster. There is an inasafe utility method that can be used to accomplish that. 
The method will return a vector layer containing one polygon. Note that `polygonize <../api-docs/safe/common/qgis_raster_tools.html#safe.common.qgis_raster_tools.polygonize>`_ function can filter the raster and eliminate some "undesired" pixels from the process. The threshold parameters represent the valid pixel values that should be considered in the process.

::  

    flooded_polygon = polygonize(small_raster, threshold_min, threshold_max)


Once we have a polygon hazard layer (and our exposure line layer), we can use the inasafe utility function `split_by_polygon <../api-docs/safe/common/qgis_vector_tools.html#safe.common.qgis_vector_tools.split_by_polygon>`_  to identify the road features that are affected. Note that the split_by_polygon will return a vector layer where elements are clipped by the polygon. The features will also contain a new field identifying them as impacted or not:

::

    # Find inundated roads, mark them
        line_layer = split_by_polygon(
            line_layer,
            flooded_polygon,
            request,
            mark_value=(target_field, 1))


The next step is to calculate the length of all impacted roads. That can be done by looping through the resulting vector layer and verifying if the feature is impacted or not. Note that to be able to properly calculate a length, It is recommended to re-project the geometry to a known projection such as UTM.

::

   # reproject features to be able to calculate lenth properly
    epsg = get_utm_epsg(self.extent[0], self.extent[1])
    output_crs = QgsCoordinateReferenceSystem(epsg)
    transform = QgsCoordinateTransform(E.crs(), output_crs)
    ....
    geom = road.geometry()
    geom.transform(transform)


|project_name| assumes that every impact function returns a raster or vector layer. The last step is to return a native safe layer
::
 
    ....
    # Convert QgsVectorLayer to inasafe layer and return it
     line_layer = Vector(data=line_layer,
                   name=tr('Flooded roads'),
                   keywords={'impact_summary': impact_summary,
                             'map_title': map_title,
                             'target_field': target_field},
                   style_info=style_info)
     return line_layer

This function is available in full at
:download:`/static/flood_raster_roads.py`



Example of an Impact function using a polygon hazard and line vector exposure
-----------------------------------------------------------------------------

In this example, the function uses a vector polygon hazard layer and a vector line features for exposure data.
The logic and concepts are similar to  :ref:`raster_line`. The only major difference is that the polygon used to clip the line features is built by combining all the polygons in the hazard layer.

::

        hazard_features = hazard.getFeatures(request)
        hazard_poly = None
        for feature in hazard_features:
            
            if hazard_poly is None:
                hazard_poly = QgsGeometry(feature.geometry())
            else:
                # Make geometry union of inundated polygons
                # But some feature.geometry() could be invalid, skip them
                tmp_geometry = hazard_poly.combine(feature.geometry())
                try:
                    if tmp_geometry.isGeosValid():
                        hazard_poly = tmp_geometry
                except AttributeError:
                    pass


This function is available in full at
:download:`/static/flood_polygon_roads.py`
