.. _operation:

Operation
=========

Nodepool has two components which run as daemons.  The
``nodepool-builder`` daemon is responsible for building diskimages and
uploading them to providers, and the ``nodepool-launcher`` daemon is
responsible for launching and deleting nodes.

Both daemons frequently re-read their configuration file after
starting to support adding or removing new images and providers, or
otherwise altering the configuration.

These daemons communicate with each other via a Zookeeper database.
You must run Zookeeper and at least one of each of these daemons to
have a functioning Nodepool installation.

Nodepool-builder
----------------

The ``nodepool-builder`` daemon builds and uploads images to
providers.  It may be run on the same or a separate host as the main
nodepool daemon.  Multiple instances of ``nodepool-builder`` may be
run on the same or separate hosts in order to speed up image builds
across many machines, or supply high-availability or redundancy.
However, since ``nodepool-builder`` allows specification of the number
of both build and upload threads, it is usually not advantageous to
run more than a single instance on one machine.  Note that while
diskimage-builder (which is responsible for building the underlying
images) generally supports executing multiple builds on a single
machine simultaneously, some of the elements it uses may not.  To be
safe, it is recommended to run a single instance of
``nodepool-builder`` on a machine, and configure that instance to run
only a single build thread (the default).


Nodepool-launcher
-----------------

The main nodepool daemon is named ``nodepool-launcher`` and is
responsible for managing cloud instances launched from the images
created and uploaded by ``nodepool-builder``.

When a new image is created and uploaded, ``nodepool-launcher`` will
immediately start using it when launching nodes (Nodepool always uses
the most recent image for a given provider in the ``ready`` state).
Nodepool will delete images if they are not the most recent or second
most recent ``ready`` images.  In other words, Nodepool will always
make sure that in addition to the current image, it keeps the previous
image around.  This way if you find that a newly created image is
problematic, you may simply delete it and Nodepool will revert to
using the previous image.

Daemon usage
------------

To start the main Nodepool daemon, run **nodepool-launcher**:

.. program-output:: nodepool-launcher --help
   :nostderr:

To start the nodepool-builder daemon, run **nodepool--builder**:

.. program-output:: nodepool-builder --help
   :nostderr:

To stop a daemon, send SIGINT to the process.

When `yappi <https://code.google.com/p/yappi/>`_ (Yet Another Python
Profiler) is available, additional functions' and threads' stats are
emitted as well. The first SIGUSR2 will enable yappi, on the second
SIGUSR2 it dumps the information collected, resets all yappi state and
stops profiling. This is to minimize the impact of yappi on a running
system.

Metadata
--------

When Nodepool creates instances, it will assign the following nova
metadata:

  groups
    A comma separated list containing the name of the image and the name
    of the provider.  This may be used by the Ansible OpenStack
    inventory plugin.

  nodepool_image_name
    The name of the image as a string.

  nodepool_provider_name
    The name of the provider as a string.

  nodepool_node_id
    The nodepool id of the node as an integer.

Common Management Tasks
-----------------------

In the course of running a Nodepool service you will find that there are
some common operations that will be performed. Like the services
themselves these are split into two groups, image management and
instance management.

Image Management
~~~~~~~~~~~~~~~~

Before Nodepool can launch any cloud instances it must have images to boot
off of. ``nodepool dib-image-list`` will show you which images are available
locally on disk. These images on disk are then uploaded to clouds,
``nodepool image-list`` will show you what images are bootable in your
various clouds.

If you need to force a new image to be built to pick up a new feature more
quickly than the normal rebuild cycle (which defaults to 24 hours) you can
manually trigger a rebuild. Using ``nodepool image-build`` you can tell
Nodepool to begin a new image build now. Note that depending on work that
the nodepool-builder is already performing this may queue the build. Check
``nodepool dib-image-list`` to see the current state of the builds. Once
the image is built it is automatically uploaded to all of the clouds
configured to use that image.

At times you may need to stop using an existing image because it is broken.
Your two major options here are to build a new image to replace the existing
image or to delete the existing image and have Nodepool fall back on using
the previous image. Rebuilding and uploading can be slow so typically the
best option is to simply ``nodepool image-delete`` the most recent image
which will cause Nodepool to fallback on using the previous image. Howevever,
if you do this without "pausing" the image it will be immediately reuploaded.
You will want to pause the image if you need to further investigate why
the image is not being built correctly. If you know the image will be built
correctly you can simple delete the built image and remove it from all clouds
which will cause it to be rebuilt using ``nodepool dib-image-delete``.

Command Line Tools
------------------

Usage
~~~~~
The general options that apply to all subcommands are:

.. program-output:: nodepool --help
   :nostderr:

The following subcommands deal with nodepool images:

dib-image-list
^^^^^^^^^^^^^^
.. program-output:: nodepool dib-image-list --help
   :nostderr:

image-list
^^^^^^^^^^
.. program-output:: nodepool image-list --help
   :nostderr:

image-build
^^^^^^^^^^^
.. program-output:: nodepool image-build --help
   :nostderr:

dib-image-delete
^^^^^^^^^^^^^^^^
.. program-output:: nodepool dib-image-delete --help
   :nostderr:

image-delete
^^^^^^^^^^^^
.. program-output:: nodepool image-delete --help
   :nostderr:

The following subcommands deal with nodepool nodes:

list
^^^^
.. program-output:: nodepool list --help
   :nostderr:

delete
^^^^^^
.. program-output:: nodepool delete --help
   :nostderr:

The following subcommands deal with ZooKeeper data management:

info
^^^^
.. program-output:: nodepool info --help
   :nostderr:

erase
^^^^^
.. program-output:: nodepool erase --help
   :nostderr:

If Nodepool's database gets out of sync with reality, the following
commands can help identify compute instances or images that are
unknown to Nodepool:

alien-image-list
^^^^^^^^^^^^^^^^
.. program-output:: nodepool alien-image-list --help
   :nostderr:

Removing a Provider
-------------------

To remove a provider, remove all of the images from that provider`s
configuration (and remove all instances of that provider from any
labels) and set that provider's max-servers to -1.  This will instruct
Nodepool to delete any images uploaded to that provider, not upload
any new ones, and stop booting new nodes on the provider.  You can
then let the nodes go through their normal lifecycle.  Once all nodes
have been deleted you remove the config from nodepool for that
provider entirely (though leaving it in this state is effectively the
same and makes it easy to turn the provider back on).

If urgency is required you can delete the nodes directly instead of
waiting for them to go through their normal lifecycle but the effect
is the same.

Web interface
-------------

If configured (see :attr:`webapp-conf`), a ``nodepool-launcher``
instance can provide a range of end-points that can provide
information in text and ``json`` format.  Note if there are multiple
launchers, all will provide the same information.

.. http:get:: /image-list

   The status of uploaded images

   :query fields: comma-separated list of fields to display
   :reqheader Accept: ``application/json`` or ``text/*``
   :resheader Content-Type: ``application/json`` or ``text/plain``
                            depending on the :http:header:`Accept` header

.. http:get:: /dib-image-list

   The status of images built by ``diskimage-builder``

   :query fields: comma-separated list of fields to display
   :reqheader Accept: ``application/json`` or ``text/*``
   :resheader Content-Type: ``application/json`` or ``text/plain``
                            depending on the :http:header:`Accept` header

.. http:get:: /node-list

   The status of currently active nodes

   :query node_id: restrict to a specific node
   :query fields: comma-separated list of fields to display
   :reqheader Accept: ``application/json`` or ``text/*``
   :resheader Content-Type: ``application/json`` or ``text/plain``
                            depending on the :http:header:`Accept` header

.. http:get:: /request-list

   Outstanding requests

   :query fields: comma-separated list of fields to display
   :reqheader Accept: ``application/json`` or ``text/*``
   :resheader Content-Type: ``application/json`` or ``text/plain``
                            depending on the :http:header:`Accept` header

.. http:get:: /label-list

   All available labels as reported by all launchers

   :query fields: comma-separated list of fields to display
   :reqheader Accept: ``application/json`` or ``text/*``
   :resheader Content-Type: ``application/json`` or ``text/plain``
                            depending on the :http:header:`Accept` header

Monitoring
----------

Nodepool provides monitoring information to statsd. See
:ref:`statsd_configuration` to learn how to enable statsd support. Currently,
these metrics are supported:

Nodepool builder
~~~~~~~~~~~~~~~~

.. zuul:stat:: nodepool.dib_image_build.<diskimage_name>.<ext>.size
   :type: gauge

   This stat reports the size of the built image in bytes.

.. zuul:stat:: nodepool.image_update.<image name>.<provider name>
   :type: counter, timer

   Number of image uploads to a specific provider in the cloud plus the time in
   seconds spent to upload the image.

.. zuul:stat:: nodepool.dib_image_build.<diskimage_name>.<ext>.rc
   :type: gauge

   Return code of the DIB.

.. zuul:stat:: nodepool.dib_image_build.<diskimage_name>.<ext>.duration
   :type: timer

   Time the DIB run took in ms

Nodepool launcher
~~~~~~~~~~~~~~~~~

.. zuul:stat:: nodepool.provider.<provider>.max_servers
   :type: gauge

   Current setting of the max-server configuration parameter for the respective
   provider.

.. _nodepool_nodes:

.. zuul:stat:: nodepool.nodes.<state>
  :type: counter

   Number of nodes in a specific state.

   state can be:

   * building
   * deleting
   * failed
   * in-use
   * ready
   * used

.. zuul:stat:: nodepool.provider.<provider>.downPorts
   :type: counter

   Number of ports in the DOWN state that have been removed automatically
   in the cleanup resources phase of the OpenStack driver.

.. zuul:stat:: nodepool.provider.<provider>.nodes.<state>
   :type: gauge

   Number of nodes per provider that are in one specific state. See
   :ref:`nodepool.nodes <nodepool_nodes>` for a list of possible states.

.. zuul:stat:: nodepool.label.<label>.nodes.<state>
   :type: counter

   Number of nodes with a specific label in a specific state. See
   :ref:`nodepool.nodes <nodepool_nodes>` for a list of possible states.

.. zuul:stat:: nodepool.task.<provider>.<task>
   :type: counter, timer

   Number of tasks executed per provider plus the duration of the task
   execution.

.. _nodepool_launch:

.. zuul:stat:: nodepool.launch.<result>
   :type: counter, timer

   Number of launches, categorized by the launch result plus the duration
   of the launch.

   *result* can be:

   * ready: launch was successful
   * error.zksession: Zookeeper session was lost
   * error.quota: Quota of the provider was reached
   * error.unknown: Some other error during launch

.. zuul:stat:: nodepool.launch.provider.<provider>.<az>.<result>
   :type: counter, timer

   Number of launches per provider per availability zone, categorized
   by the launch result plus duration of the launch.

   See :ref:`nodepool.launch <nodepool_launch>` for a list of possible results.

.. zuul:stat:: nodepool.launch.image.<image>.<result>
   :type: counter, timer

   Number of launches per image, categorized by the launch result plus duration
   of the launch.

   See :ref:`nodepool.launch <nodepool_launch>` for a list of possible results.

.. zuul:stat:: nodepool.launch.requestor.<requestor>.<result>
   :type: counter, timer

   Number of launches per requestor, categorized by the launch result plus the
   duration of the launch.

   See :ref:`nodepool.launch <nodepool_launch>` for a list of possible results.

OpenStack API stats
~~~~~~~~~~~~~~~~~~~

Low level details on the timing of OpenStack API calls will be logged
by the API task manager.  These calls are logged under
``nodepool.task.<provider>.<api-call>``.  The API call name is of the
generic format ``<endpoint><method><operation>`` transformed into a
CamelCase value with no deliminators; for example the
``compute.GET.servers`` call becomes ``ComputeGetServers`` and
``compute.POST.os-volumes_boot`` becomes ``ComputePostOsVolumesBoot``.

Since these calls reflect the internal operations of the
``openstacksdk``, the exact keys logged may vary across providers and
releases.
