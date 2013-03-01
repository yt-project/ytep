YTEP-0000: Source Rendering 
===========================

To write a YTEP, copy this template to the next numerical number, add it to the
repository, and issue a pull request.  Discussion of the YTEP will occur either
on the mailing list (for large-scale changes) or in the PR itself (small items,
such as formatting).

This document has been patterned after the Matplotlib Enhancement Proposal
Template (found `here
<https://github.com/matplotlib/matplotlib/wiki/MEPTemplate>`_.)

Abstract
--------

Created: February 28, 2012
Author: Sam Skillman

This proposal outlines the changes (functional and API) necessary for the 
ability to render volumes using arbitrary data sources.  This will still
operate with the idea of grids and masks.  However, this should lead as a 
stepping stone to non-grid-based rendering.

Status
------

Status should be one of the following:

 #. In Progress

YTEPs do not need to pass through every stage.

Project Management Links
------------------------

Currently the development of this capability is in the camera-refactor
bookmark at:
https://bitbucket.org/samskillman/yt/commits/all/tip/..bookmark%28%22camera-refactor%22%29

Detailed Description
--------------------

Functional Background:

The volume rendering has long operated on either the entire domain or (at best)
a sub-rectangular-prism using left and right edges.  This is fairly limiting in
that a user may not need (or want) to render the entire volume, and may want
to restrict the volume shown by something other than a single box.  A majority
of the reasoning behind the recent ``AMRKDTree`` was to allow for more generic
adding of grids/data to the homogenized volume.

The primary problem with attempting to do this is that the volume rendering
acts on a rectangular brick of data, and the traversal of this brick is fairly 
far removed from a AMR3DData object.  Therefore, we either need to modify the 
traversal, or somehow mask out the data being handed to the traversal.

This proposal takes the latter approach by creating a mask that modifies the
data being passed to the integrator.

I have taken a first pass at implementing this which is done by creating
a 'source_mask' derived field based of the source._get_cut_mask() function.
There may be a better way to do this, but this seems to work.  By using this
derived field to create a mask, we then take all points outside the data object
and multiply the incoming data (Density, Temperature, etc), by np.inf.  No 
physical field should be np.inf, where some fields may easily be very large,
very small, or zero.  This seems to function as expected, and the transfer
function is zero for np.inf. 

There are several stumbling points:

  * no_ghost=False does not seem to work as the smoothed covering grid doesn't
    like the cut_mask as being what is used to generate the field.
  * Since the source_mask is smoothed, it may non-trivially affect the data
    values at the edges of the source. I haven't seen this show up yet, but I 
    wouldn't be surprised to see it.

API Background:

The method for specifying a data source in a volume rendering has not been
suggested.  Currently, the rectangular volume that is used for rendering uses
a ``le`` and ``re`` pair of keywords to specify the left and right edges.  When
moving to a data source, we should simplify the volume selection by simply
supplying a ``source=`` (or ``data_source=``) keyword.  In order to make the 
process more clear, this YTEP also proposes getting rid of many of the
unnecessary arguments and keyword arguments to the camera constructor.

In my working solution, I have set the AMRKDTree __init__ function to have the
following form:

```
#!python
class AMRKDTree(ParallelAnalysisInterface)
    def __init__(self, pf, min_level=None, max_level=None, source=None):

```

Similarly, the current constructor for the Camera object has been changed to:

```
#!python
class Camera(...):
    def __init__(self, center, normal_vector, width,
                 resolution, transfer_function,
                 north_vector = None, steady_north=False,
                 volume = None, fields = None,
                 sub_samples = 5, pf = None,
                 min_level=None, max_level=None, no_ghost=True,
                 source=None,
                 use_light=False):

```

I would like to propose that the Camera constructor eventually take the form:

```
#!python
class Camera(...):
    def __init__(self, pf, resolution, fields=None, source=None):

```

If ``source`` is none, it will default to pf.h.all_data. Fields would no longer
default to Density, and instead must be specified at construction or later with
a set_field() method. I would even suggest resolution be set at 256/512 with,
again, a helper function to modify it if requested.  If not specified, an 
onion-peel transfer function would be constructed, as is done in the yt
rendering command line interface.  The tf can then be modified (as is already
possible) with cam.transfer_fuction.clear() and
cam.transfer_function.add_layers(...), etc.   

This would greatly simplify the interface to the camera, allowing users to get
a first look without specifying a center, width, L, etc.  

Backwards Compatibility
-----------------------

This YTEP breaks the following backwards compatibility:

  * Camera API
  * AMRKDTree API

It will additionally break internal uses of the API for the Camera, other
cameras inheriting the __init__ of Camera, and the AMRKDTree.


Alternatives
------------

  * Do nothing
  * Add more keyword arguments to everything
  * Wait until rendering is ready in yt-3.0, which will also likely demand
    a breakage of API.
  * The Camera could be left as is, but the creation of a different VR
    framework such as a "Scene" could be implemented from the ground up.

My only reasoning for breaking things for the 2.6+ release is that it is
a simplification and I've already started using it with great success.  I'm
also hopeful that this simplification is along the same lines of the idea of
a simplified volume rendering scene object.
