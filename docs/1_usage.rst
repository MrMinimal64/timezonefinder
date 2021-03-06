.. _usage:

=====
Usage
=====

.. note::

   Also check out the :ref:`API documentation <api>` or the `code <https://github.com/MrMinimal64/timezonefinder>`__.


.. _init:

Initialisation
--------------


Create a new instance of the :ref:`TimezoneFinder class <api_finder>` to allow fast consequent timezone queries:

.. code-block:: python

    from timezonefinder import TimezoneFinder

    tf = TimezoneFinder()


To save computing time at the cost of memory consumption and initialisation time pass ``in_memory=True``. This causes all binary files to be read into memory.
See the :ref:`speed test results <speed-tests>`.

.. code-block:: python

    tf = TimezoneFinder(in_memory=True)


Use the argument ``bin_file_location`` to use data files from another location (e.g. :ref:`your own compiled files <parse_data>`):

.. code-block:: python

    tf = TimezoneFinder(bin_file_location="path/to/files")





For testing if the import of the JIT compiled algorithms worked:


.. code-block:: python

    TimezoneFinder.using_numba()  # returns True or False



timezone_at()
--------------

This is the default function to check which timezone a point lies within.
If no timezone has been matched, ``None`` is being returned.



.. code-block:: python

    latitude, longitude = 52.5061, 13.358
    tf.timezone_at(lng=longitude, lat=latitude)  # returns 'Europe/Berlin'

.. note::
    * to avoid mixing up the arguments latitude and longitude have to be given as keyword arguments
    * this function is optimized for speed and the common case to only query points within a timezone. The last possible timezone in proximity is always returned (without checking if the point is really included). So results might be misleading for points outside of any timezone.



timezone_at_land()
------------------

This package includes ocean timezones (``Etc/GMT...``).
If you want to explicitly receive only "land" timezones use

.. code-block:: python

    tf.timezone_at_land(lng=longitude, lat=latitude)  # returns 'Europe/Berlin'



unique_timezone_at()
--------------------

For fast execution ``timezonefinder`` internally uses precomputed "shortcuts" which store the possible zones in proximity.
Call ``unique_timezone_at()`` if you want to ask for an exact result without actually performing "point-in-polygon" tests (<- computationally expensive).
This function will return ``None`` when the correct zone cannot be uniquely determined without further computation.

.. code-block:: python

    tf.unique_timezone_at(lng=longitude, lat=latitude)



.. note::
    The "lightweight" class :ref:`TimezoneFinderL <usage_finderL>`, which is using only shortcuts, also supports just querying the most probable timezone.


certain_timezone_at()
----------------------

.. note::

    DEPRECATED: Due to the included ocean timezones one zone will always be matched.
    Use ``timezone_at()`` or ``timezone_at_land()`` instead.


This function is for making sure a point is really inside a timezone. It is slower, because all polygons (with shortcuts in that area)
are being checked until one polygon is matched. ``None`` is being returned in the case of no match.


.. code-block:: python

    tf.certain_timezone_at(lng=longitude, lat=latitude)  # returns 'Europe/Berlin'



.. note::

    Due to the "point-in-polygon-test" algorithm being used, the state of a point on the edge of a (timezone) polygon is undefined.
    For those kind of points the return values is hence uncertain and might be ``None``.
    This applies for example for all points with lng=+-180.0, because the timezone polygons in the data set are being cropped at the 180 longitude border.



closest_timezone_at()
----------------------


This function computes and compares the distances to the timezone polygon boundaries (expensive!).
By default the function returns the closest timezone of all polygons within +-1 degree lng and +-1 degree lat (or None).



.. code-block:: python

    longitude = 12.773955
    latitude = 55.578595
    tf.closest_timezone_at(lng=longitude, lat=latitude)  # returns 'Europe/Copenhagen'



.. note::

    * This function does not check whether a point is included in a timezone polygon.
    * The timezone polygons do NOT follow the shoreline. This causes the computed distance from a timezone polygon to be not really accurate!



**Options:**


To increase search radius even more, use the ``delta_degree``-option:

.. code-block:: python

    tf.closest_timezone_at(lng=longitude, lat=latitude, delta_degree=3)


This checks all the polygons within +-3 degree lng and +-3 degree lat.
I recommend only slowly increasing the search radius, since computation time increases quite quickly
(with the amount of polygons which need to be evaluated). When you want to use this feature a lot,
consider using ``Numba`` to save computing time.


.. note::

    x degrees lat are not the same distance apart than x degree lng (earth is a sphere)!
    As a consequence getting a result does NOT mean that there is no closer timezone! It might just not be within the area (given in degree!) being queried.


With ``exact_computation=True`` the distance to every polygon edge is computed (way more complicated), instead of just evaluating the distances to all the vertices.
This only makes a real difference when the boundary of a polygon is very close to the query point.


With ``return_distances=True`` the output looks like this:

::

    ( 'tz_name_of_the_closest_polygon',[ distances to every polygon in km], [tz_names of every polygon])


.. note::

    Some polygons might not be tested (for example when a zone is found to be the closest already).
    To prevent this use ``force_evaluation=True``.


A single timezone might be represented by multiple polygons and the distance to each of the candidate polygons is being computed and returned. Hence one may get multiple results for one timezone. Example:


.. code-block:: python

    longitude = 42.1052479
    latitude = -16.622686
    tf.closest_timezone_at(
        lng=longitude,
        lat=latitude,
        delta_degree=2,
        exact_computation=True,
        return_distances=True,
        force_evaluation=True,
    )
    """
    returns ('uninhabited',
    [80.66907784731714, 217.10924866254518, 293.5467252349301, 304.5274937839159, 238.18462606485667, 267.918674688949, 207.43831938964408, 209.6790144988553, 228.42135641542546],
    ['uninhabited', 'Indian/Antananarivo', 'Indian/Antananarivo', 'Indian/Antananarivo', 'Africa/Maputo', 'Africa/Maputo', 'Africa/Maputo', 'Africa/Maputo', 'Africa/Maputo'])
    """



get_geometry()
--------------


For querying a timezone for its geometric multi-polygon shape use ``get_geometry()``.
output format: ``[ [polygon1, hole1,...), [polygon2, ...], ...]``
and each polygon and hole is itself formated like: ``([longitudes], [latitudes])``
or ``[(lng1,lat1), (lng2,lat2),...]`` if ``coords_as_pairs=True``.


.. code-block:: python

    tf.get_geometry(tz_name="Africa/Addis_Ababa", coords_as_pairs=True)
    tf.get_geometry(tz_id=400, use_id=True)




.. _usage_finderL:

TimezoneFinderL
---------------

:ref:`TimezoneFinderL <api_finderL>` is a light version of the :ref:`TimezoneFinder class <api_finder>`.
It is useful for quickly suggesting probable timezones without using as many computational resources (cf. :ref:`speed tests <speed-tests>`).
Instead of using timezone polygon data this class instantly returns the timezone just based on precomputed "shortcuts".

Check the (:ref:`API documentation <api_finderL>`) of ``TimezoneFinderL``.

The most probable zone in proximity can be retrieved with ``timezone_at()``:

.. code-block:: python

    from timezonefinder import TimezoneFinderL

    tf = TimezoneFinderL(in_memory=True)
    latitude, longitude = 52.5061, 13.358
    tf.timezone_at(lng=longitude, lat=latitude)  # returns 'Europe/Berlin'



Certain results can be retrieved with ``unique_timezone_at()``:

.. code-block:: python

    tf.unique_timezone_at(lng=longitude, lat=latitude)  # returns 'Europe/Berlin'


.. note::

    If you only use ``TimezoneFinderL``, you may delete all data files except ``timezone_names.json``, ``shortcuts_unique_id.bin`` and ``shortcuts_direct_id.bin`` to obtain a truly lightweight installation.




Using vectorized input
----------------------

Check `numpy.vectorize <https://docs.scipy.org/doc/numpy/reference/generated/numpy.vectorize.html>`__
and `pandas.DataFrame.apply <https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.apply.html>`__



Calling timezonefinder from the command line
---------------------------------------------

A command line script is being installed as part of this package.

**Command Line Syntax**:

::

    timezonefinder [-h] [-v] [-f {0,1,2,3,4,5}] lng lat


**Example**:

::

    timezonefinder -f 4 40.5 11.7


With ``-v`` you get verbose output, without it only the timezone name is being printed.
With the argument of the flag ``-f`` one can choose between the different functions to be called:

::

    0: TimezoneFinder.timezone_at() = default
    1: TimezoneFinder.certain_timezone_at()
    2: TimezoneFinder.closest_timezone_at()
    3: TimezoneFinderL.timezone_at()
    4: TimezoneFinderL.timezone_at_land()
    5: TimezoneFinder.timezone_at_land()


Please note that this will be orders of magnitude slower than using the package directly from within python.
