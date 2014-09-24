..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Glance Swift Store to use Multiple Containers for Storing Images
==========================================

Glance, when configured to use Swift store in Single Tentant Mode, stores
images in one container as indicated by the configuration option,
swift_store_container. This approach of storing images in ONE container
is subject to a) a higher risk of data loss and b) performance bottleneck.

Data Loss:
 As Glance stores all images in ONE container it is particularly prone to
 complete data loss, should the container be deleted accidentally or 
 maliciously.

Performance Bottleneck:
 Storing images in one container may also be prone to Swift rate-limiting on
 containers. Swift is equipped with container rate-limiting that can throttle
 concurrent POST, PUT and DELETE operations. 


Problem description
===================

Using one container to store all images presents a clear case of losing all
data in case of an accidental deletion. On the other hand, Swift is known to
be capable of throttling incoming traffic[1]. Though this varies from 
deployment to deployment, the very fact that swift can throttle write
operations on containers presents a performance bottleneck and hence some
deployments may need an alternative to get around Swift throttling.

When container rate-limiting is enabled for a Swift cluster, it throttles
concurrent POST, PUT and DELETE requests after a certain configurable rate.
This directly translates to a limit on concurrent image creation and deletion
operations for Glance before experiencing performance degradation.


Proposed change
===============
To reduce/overcome the risk of data loss and performance bottleneck, we propose
the use of multiple containers for storing images in Single Tenant Mode (this 
change will not affect Multi Tentant Mode because that setup stores each image
in its own container). This leads to reduced risk of data loss and increased
concurrency of image creation and deletion operations at usual performance.

There are four major aspects to this change:
* container selection - determining what container an image should go into
* container creation - creating the new containers
* re-distribution of existing images - moving images from old to new containers
* database migration - updating image locations as per new containers

Container Selection:
This change proposes to select containers based on image uuid. Images will be 
stored in multiple containers in order to avoid throttling during 
multiple simultaneous uploads. The first N characters of the image UUID (where
N is a configurable integer between 1 and 32 with the default value of 3) will
be used to determine which container the image will be uploaded to. With the 
default value of the first three characters used, this gives 16*16*16=4096 
unique containers. At N=1, the smallest valid value for this configuration, 16
containers will be created and used for storing images. The name for these new 
containers will be the swift_store_container + _ + first N characters of image
UUID. Example: glance_3aF for image UUID 3AF1B5D.... with 
swift_store_container = 'glance' and swift_store_multiple_containers_seed = 3

The number of containers can be easily increased or decreased by changing N in
the configuration. However, a new set of containers will be created with every
change to this configuration. This leads to images created after the
configuration change go into new containers while older images remain in older
containers. The older images need not necessarily be moved into new containers
as their locations would still point to the older container they are stored in.
However, if one wishes to move the older images to new containers, they may do
so by re-distributing the images, which is described later in this section.


Container Creation:
Glance ships with a configuration option to dynamically create the container,
if it doesn't exist already at the time of uploading image data to Swift. This
is indicated by configuration option, swift_store_create_container_on_put. If
dynamic container creation is enabled, Glance would automatically create all
required containers, 4096 by default, when it hits the first image for each
combination of first three characters of a UUID.

However, if dynamic container creation is disabled, image uploads would fail
citing a missing container as the error. In such a case, appropriate containers
need to created beforehand. This behavior is consistent with how Glance
currently behaves if the image's container doesn't exist.


Re-distribution of Existing Images:
Once the use of multiple containers is enabled or the number of containers is
changed, all previously created images would remain in the older container(s).
If desired, older images can be moved to new containers appropriately. This can
be achieved as a separate batch job that can be run as and when desired.
Subject to the number of older images, redistributing images may involve
significant movement of data in the Swift cluster. Hence, it would be helpful
to achieve this in phases and in a non-intrusive fashion.

Once the images are re-distributed, their image locations need to be updated.


Database Migration:
If images are re-distributed, image location of each re-distributed image must
be updated to reflect the new container name. This requires a db migration to
replace the old container name in the location with the new container name as
per the image id. This migration can go hand-in-hand with re-distribution.



Of the above four aspects discussed above, this change addresses container
creation and selection in particular while leaving re-distribution and the
required db migrations out, which can be implemented as another concerted
effort.


Alternatives
------------

* Instead of using image id as the basis for container selection, one can use
other basis like tenant id, which would keep all images belonging to a certain
tenant in the same container. While there are may be many other bases possible,
using image id provides an easier way to correlate an image to its container.

* An alternative to creating containers could be to allow the API to create all
the required containers while it boots up. This requires the API to know all
possible containers before hand, which may or may not be possible depending
upon the container selection basis chosen. This places a certain limitation on
the kind of bases one may opt for. Hence, going with dynamic container creation
will eliminate this limitation as both container selection and creation could
be dynamic. Also, dynamic container creation is in-line with current Glance 
behavior.

Data model impact
-----------------
New containers will be created and used for storing images. However, this
should have any impact on the Glance image data model itself.


Database migrations: No database migrations are required. The code 
supporting multiple containers would only affect the uploading of new images,
determining which container they belong to based on uuid. For existing images
(those uploaded before support for multiple containers), the image already
contains a valid location in its metadata. Essentially, new containers will
be populated by lazy loading: When an image is uploading, it will first check
if the appropriate container exists for that image based on its UUID, and if
not then the container will be created on the spot, then that image will be
the first image stored in that container.
 

REST API impact
---------------

None

Security impact
---------------
Given the scope of this spec, where image data is not being re-distributed 
among new containers and no migrations are being run, there is minimal
to no security impact introduced. 

Moreover, this change reduces the risk of data loss since images will be 
spread across multiple containers. If one container is deleted (accidently or
maliciously) it has much less of an impact compared to the single container
model. Also, since placement of images in these containers is deterministic, 
it is feasible to manually re-create containers or the manifest if they are 
deleted.

Notifications impact
--------------------

This change only impacts the image location property among all the image 
properties. And, since image location is not included in notifications, there 
should be no impact to Glance notifications.

Other end user impact
---------------------

As image location is not accessible to either the end-user or from Glance 
client, there should be no end-user impact.

Performance Impact
------------------
* Container selection would take place for every image upload request and thus
adds an extra operation to the current set of operations to upload image data.
However, selecting a container would be a simple substring operation to fetch
the first few characters of an image id. The time incurred in determining the
container would be significantly smaller than the time incurred to upload image
data. Overall, the performance impact of container selection should be very
minimal.

* Container creation is a conditional operation that would take place only when
the container is not present already. This would occur once for each 
combination of N characters as specified in the configuration. 
For example: when N=3, container creation occurs 4096 times, once per each 
combination of first three characters of a UUID. Also, the time incurred in 
creating a new container is expected to be significantly smaller than the time 
incurred in upload image data. Hence, the overall performance impact in image 
uploads should be minimal.  

* On the other hand, the use of multiple containers will reduce 
throttling when multiple images are uploaded simultaneously. 
This leads to increased concurrency of image creation and deletion
 operations


Other deployer impact
---------------------

This change would begin taking effect upon enabling multiple 
containers in a configuration. When enabled, new images would 
be uploaded to new containers, while existing images would 
remain in their previously assigned container.


New configuration options in glance-api.conf:

swift_store_use_multiple_containers - default value = False 
- A boolean value that determines if single-tenant 
store should use multiple containers for storing images. 
Used only when swift_store_multi_tenant is disabled.

swift_store_multiple_containers_seed - default value=  3 
- An integer value between 1 and 32 representing the number of 
characters used from the image UUID to determine which container
the image will be placed. Used only when 
swift_store_use_multiple_containers is enabled. The total number 
of containers that will be used is equal to 16^N, or 16^3=4096 
by default.



Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  hemanth-makkapati

Other contributors:
  ben-roble

Work Items
----------

1) Implement new config options in Swift store driver
2) Implement container selection in Swift store driver
3) Implement unit, functional, and integration tests
4) Change glance-api sample conf in glance repo

Points to note:
* All code changes would be limited to glance_store module.
* Image download code wouldn't require any changes.
* Both manifest and segments would go into the same container.

Dependencies
============

None


Testing
=======

No tempest tests needed


Documentation Impact
====================

* Document new configuration options

References
==========

[1] http://docs.openstack.org/developer/swift/ratelimit.html#configuration
