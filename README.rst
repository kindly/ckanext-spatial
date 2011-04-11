==============================================
ckanext-spatial - Geo related plugins for CKAN
==============================================

This extension contains plugins that add geospatial capabilities to CKAN.
Currently, there are a WMS previewer (`wms_preview`) and a spatial query
API call (`spatial_query`) available.

Dependencies
============

You will need CKAN installed. The present module should be installed at least 
with `setup.py develop` if not installed in the normal way with
`setup.py install` or using pip or easy_install.
 
If you want to use the spatial search API, you will need PostGIS installed
and enable the spatial features of your PostgreSQL database. See the
"Setting up PostGIS" section for details.

Configuration
=============

Plugins are configured as follows in the CKAN ini file::

    ckan.plugins = wms_preview spatial_query

If you are using the spatial search feature, you can define the projection
in which extents are stored in the database with the following option. Use 
the EPSG code as an integer (e.g 4326, 4258, 27700, etc). It defaults to 
4258::
    
    ckan.spatial.srid = 4258


Command line interface
======================

The following operations can be run from the command line using the 
``paster extents`` command::

      extents update 
         - creates or updates the extent geometry column for packages with
          a bounding box defined in extras
       
The commands should be run from the ckanext-spatial directory and expect
a development.ini file to be present. Most of the time you will specify 
the config explicitly though::

        paster extents update --config=../ckan/development.ini


API
===

The extension adds the following call to the CKAN search API, which returns
packages with an extent that intersects with the bounding box provided::

    /api/2/search/package/geo?bbox={minx,miny,maxx,maxy}[&crs={srid}]

If the bounding box coordinates are not in the same projection as the one
defined in the database, a CRS must be provided, in one of the following
forms:

- urn:ogc:def:crs:EPSG::4258
- EPSG:4258
- 4258


Setting up PostGIS
==================

Configuration
-------------

*   Install PostGIS::

        sudo apt-get install postgresql-8.4-postgis
    
*   Create a new PostgreSQL database::
    
        sudo -u postgres createdb [database]
        
    (If you just want to spatially enable an exisiting database, you can
    ignore this point, but it's a good idea to create a template to
    make easier to create new databases)

*   Many of the PostGIS functions are written in the PL/pgSQL language,
    so we need to enable it in our database::
    
        sudo -u postgres createlang plpgsql [database]

*   Run the following commands. The first one will create the necessary
    tables and functions in the database, and the second will populate
    the spatial reference table::
    
        sudo -u postgres psql -d [database] -f /usr/share/postgresql/8.4/contrib/postgis-1.5/postgis.sql
        sudo -u postgres psql -d [database] -f /usr/share/postgresql/8.4/contrib/postgis-1.5/spatial_ref_sys.sql    

*   Execute the following command to see if PostGIS was properly
    installed::
    
        sudo -u postgres psql -d [database] -c "SELECT postgis_full_version()"
    
    You should get something like::
    
                                             postgis_full_version                                          
        ------------------------------------------------------------------------------------------------------
        POSTGIS="1.5.2" GEOS="3.2.2-CAPI-1.6.2" PROJ="Rel. 4.7.1, 23 September 2009" LIBXML="2.7.7" USE_STATS
        (1 row)
    
    Also, if you log into the database, you should see two tables,
    ``geometry_columns`` and ``spatial_ref_sys`` (and probably a view
    called ``geography_columns``).

    Note: This commands will create the two tables owned by the postgres
    user. You probably should make owner the user that will access the
    database from ckan::
    
        ALTER TABLE spatial_ref_sys OWNER TO [your_user];
        ALTER TABLE geometry_columns OWNER TO [your_user];

More information on PostGIS installation can be found here:

http://postgis.refractions.net/docs/ch02.html#PGInstall



Setting up a spatial table
--------------------------

To be able to store geometries and perform spatial operations, PostGIS
needs to work with geometry fields. Geometry fields should always be
added via the ``AddGeometryColumn`` function::

    CREATE TABLE package_extent(
        package_id text PRIMARY KEY
    );
    
    ALTER TABLE package_extent OWNER TO [your_user];
    
    SELECT AddGeometryColumn('package_extent','the_geom', 4258, 'POLYGON', 2);
    
This will add a geometry column in the ``package_extent`` table called
``the_geom``, with the spatial reference system EPSG:4258. The stored 
geometries will be polygons, with 2 dimensions.

Have a look a the table definition, and see how PostGIS has created
three constraints to ensure that the geometries follow the parameters
defined in the geometry column creation::

    # \d package_extent

       Table "public.package_extent"
       Column   |   Type   | Modifiers 
    ------------+----------+-----------
     package_id | text     | not null
     the_geom   | geometry | 
    Indexes:
        "package_extent_pkey" PRIMARY KEY, btree (package_id)
    Check constraints:
        "enforce_dims_the_geom" CHECK (st_ndims(the_geom) = 2)
        "enforce_geotype_the_geom" CHECK (geometrytype(the_geom) = 'POLYGON'::text OR the_geom IS NULL)
        "enforce_srid_the_geom" CHECK (st_srid(the_geom) = 4258)