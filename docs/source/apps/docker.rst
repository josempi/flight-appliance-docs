.. _docker:

Using Gridware with Docker
##########################

Gridware can now be transported within a docker container which can be shared with the nodes in a Flight environment. This allows applications to be packaged within in a container for distribution between nodes or even to systems outside of the Flight environment. 

To install docker support, apply the feature profile (for more info on features see :ref:`feature-profiles`)::

    alces customize apply feature/configure-docker

.. note:: The feature profile will install the required docker dependencies and perform Flight specific configuration changes. Any systems within the cluster that are to use docker will need to apply the profile as well

Viewing Images
==============

The ``alces gridware docker`` command can be used to interact with docker images & containers once the feature has been applied to the system.

To show available docker images::

    alces gridware docker list

On a fresh build there won't be any ``Local`` images but once the first build is run, the ``base`` image along with whatever apps have been built into containers will appear in the list. 

.. _docker-build-images:

Building Images
===============

The image ``base`` from ``docker.io/alces`` will be used to build new images with the chosen gridware application. 

To build a gridware application as a docker image::

    alces gridware docker build apps/memtester/4.3.0
    
.. note:: Unlike standard gridware, this requires the version number to be specified even if there aren't multiple versions available

This will download the base image & run the additional commands required to install the application into the image. Once this has completed then the image will be visible in the ``Local`` image list.::

    Local:
      apps-memtester-4.3.0
      base

Multiple Apps in Image
----------------------

Multiple gridware apps can be packaged into a docker image, this is done by simply appending the rest of the apps to the build command (``--name`` is specified to reduce the length and complexity of the output image name)::

    alces gridware docker build --name my-super-apps apps/memtester/4.3.0 mpi/openmpi/1.10.2 apps/gnuplot/5.0.2

The resulting image can be interacted with in the same way as a single app gridware container.

Running Containers
==================

When an image has been created it will only get started when it is interacted with using ``alces gridware docker run`` which launches a container built from the image. The containers can be interacted with in a couple of ways using this command, these methods are outlined below.

Running a Single Command
------------------------

Provide a single command as an argument to the run command after specifying the container to launch::

    [alces@login1(scooby) ~]$ alces gridware docker run apps-memtester-4.3.0 uptime
    Executing 'alces/gridware-apps-memtester-4.3.0' with arguments 'uptime'...

      >>>  15:46:02 up  1:40,  0 users,  load average: 0.20, 0.26, 0.23

    Job completed successfu8lly.

    Output summary:

    /home/alces/apps-memtester-4.3.0/work.6ff32278-2f4e-11e7-b988-0a6424937e45/output
      total 0
      drwx------ 2 alces alces  6 May  2 15:45 .
      drwx------ 3 alces alces 36 May  2 15:45 ..
      
.. note:: Any input files will need to be copied to ``~/app-memtester-4.3.0/input/`` to appear at ``/job/input/`` within the container. (Replace app-memtester-4.3.0 with the name of the container that gridware created)

Running a Script
----------------

Local scripts can be passed through to the container which allows for multiple commands to be ran at a time. Take the below script, ``~/script.sh``, which runs a few quick commands::

    #!/bin/bash
    uptime
    free -m
    echo $HOSTNAME 

This can be passed to a container using docker run as follows::

    [alces@login1(scooby) ~]$ alces gridware docker run apps-paraview-4.3.1 --script script.sh 
    Executing script 'script.sh' in 'alces/gridware-apps-paraview-4.3.1'...

      >>>  16:02:02 up  1:56,  0 users,  load average: 3.11, 2.40, 1.39
      >>>               total        used        free      shared  buff/cache   available
      >>> Mem:           7565         728         159          17        6676        6466
      >>> Swap:             0           0           0
      >>> 7ea913ce82a3

    Job completed successfully.

    Output summary:

    /home/alces/apps-paraview-4.3.1/work.ab7a1250-2f50-11e7-85da-0a6424937e45/output
      total 0
      drwx------ 2 alces alces  6 May  2 16:01 .
      drwx------ 3 alces alces 54 May  2 16:01 ..

Running in Parallel
-------------------

The argument ``--mpi=X`` can be appended to the run command to distribute the container across X processors. A 4 core example script is below::

    [alces@login1(scooby) ~]$ alces gridware docker run --mpi=4 apps-memtester-4.3.0 memtester 1G 1

Additionally, the following files are created within the container:

  - ``/job/work/hosts`` - This contains a hosts file with entries for each slave container that is running as part of the job.
  - ``/job/work/hostlist`` - As above but containing the IP addresses of the slave containers.

The MPI argument *can only* be used from the master (login) node. It will then act as the master and allocate cores on any client nodes that have the 'configure-docker' feature installed. 

Other Running Options
---------------------

Without Gridware Entrypoint
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The Gridware entrypoint is used as a standard for executing commands inside the container. The script loads the modules for the applications installed within the container and handles running the applications. In the event of wanting to run a command without using the Gridware launched, run it as follows::

    alces gridware docker run apps-memtester-4.3.0 --command uptime

Interactively
^^^^^^^^^^^^^

Instead of running the container ephemerally for a single command/script, an interactive session can be launched::

    alces gridware docker run apps-memtester-4.3.0 --interactive

Data Storage and Images
=======================

Gridware docker creates ``~/CONTAINER-NAME/`` when a container is launched with ``alces gridware docker run``, this directory contains:

  - ``input/`` - Any files in here are mapped to ``/job/input/`` in the container instance
  - ``work.CONTAINER-UUID`` - This directory is unique to the container instance and contains:

    - ``Dockerfile`` - The recipe file used to build the container
    - ``docker.log`` - Output from the building of the container
    - ``output/`` - Any files saved to ``/job/output/`` within the container will appear here
    - Any files written to ``/job/work/`` will appear in this directory

The working directory can be changed from ``~/CONTAINER-NAME`` to a user-specified directory with the ``workdir`` flag when launching the container. The directory will be created if not present and the hierarchy of the directory will be the same as listed above::

    alces gridware docker run --workdir ~/my_container_directory/ apps-memtester-4.3.0 touch /job/output/testfile

Adding Extra Mountpoints
------------------------

Directories from the host system can be mounted within the container by using the ``mount`` option when running a container::

    alces gridware docker run apps-memtester-4.3.0 --mount ~/my_local_directory/:/container_mnt ls /container_mnt

.. _docker-share-images:

Sharing Images
==============

In order for nodes to be able to use the same container that was built on the login node it will need to be shared, this can be done either through the share feature or a local registry.

Share Feature
-------------

Run the following command to add the local image to an NFS share that can be seen by the nodes::

    alces gridware docker share apps-memtester-4.3.0

Once shared, the nodes will automatically make the image available locally.

.. note:: It can take a few minutes for the image to be available on the nodes

Local Registry
--------------

A local docker registry allows for the pushing and pulling of docker images. Much like docker.io images but without the requirement of upstream connections for sharing. To start the local registry::

    alces gridware docker start-registry

Once the registry is running it can then be interacted with with ``docker push`` and ``docker pull`` as per the `docker documentation <https://docs.docker.com/registry/deploying/#copy-an-image-from-docker-hub-to-your-registry>`_

.. note:: Any other systems that are to use the docker containers will need the docker feature enabled with ``alces customize apply feature/configure-docker``
