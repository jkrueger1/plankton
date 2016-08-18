|Build Status| |Coverage Status|

Plankton
========

Plankton is a Python framework for simulating hardware devices. It is
compatible with both Python 2 and 3.

Plankton can be run directly using Python 2.7 or 3.x, or using a
prepackaged Docker image that includes all dependencies. See relevant
usage sections for details.

Resources:

- `GitHub <https://github.com/DMSC-Instrument-Data/plankton>`__
- `DockerHub <https://hub.docker.com/r/dmscid/plankton/>`__
- `Dockerfile <https://github.com/DMSC-Instrument-Data/plankton/blob/master/Dockerfile>`__

Purpose and Use Cases
---------------------

Plankton is being developed in the context of instrument control at the
`ESS <http://europeanspallationsource.se>`__, but it is general enough
to be used in many other contexts that require detailed, stateful
software simulations of hardware devices.

We consider a detailed device simulation to be one that can communicate
using the same protocol as the real device, and that can very closely
approximate real device behaviour in terms of what is seen through this
protocol. This includes gradual processes, side-effects and error
conditions.

The purpose of Plankton is to provide a common framework to facilitate
the development of such simulators. The framework provides a common set
of tools and abstracts away protocol adapters, which helps minimize code
replication and allows the developer of a simulated device to focus on
capturing device behaviour.

Potential use cases for detailed device simulators include:

-  Replacing the physical device when developing and testing software
   that interfaces with the device
-  Testing failure conditions without risking damage to the physical
   device
-  Automated system and unit tests of software that communicates with
   the device
-  Perform "dry runs" against test scripts that are to be run on the
   real device

Using a simulation for the above has the added benefit that, unlike most
real devices, a simulation may be sped up / fast-forwarded past any
lengthy delays or processes that occur in the device.

Framework Details
-----------------

The Plankton framework is built around a cycle-based statemachine that
drives the device simulation, and shared protocol adapters that separate
the communication layer from the simulated device.

Cycle-based
^^^^^^^^^^^

By cycle-based we mean that all processing in the framework occurs
during "heartbeat" simulation ticks that propagate calls to ``process``
methods throughout the simulation, along with a Δt (delta t) parameter
that contains the time that has passed since the last tick. The device
simulation is then responsible for updating its state based on how much
time has passed and what input has been received during that time.

The benefits of this approach include:

-  This closely models real device behaviour, since processing in
   electronic devices naturally occurs on a cycle basis.
-  As a side-effect of the above, certain quirks of real devices are
   often captured by the simulated device naturally, without additional
   effort.
-  The simulation becomes deterministic: The same amount of process
   cycles, with the same Δt parameters along the way, and the same input
   via the device protocol, will always result in exactly the same
   device state.
-  Simulation speed can be controlled by increasing (fast-forward) or
   decreasing (slow-motion) the Δt parameter by a given factor.
-  Simulation fidelity can be controlled independently from speed by
   increasing or decreasing the number of cycles per second while
   adjusting the Δt parameter to compensate.

The above traits are very desirable both for running automated tests
against the simulation, and for debugging any issues that are
identified.

Statemachine
^^^^^^^^^^^^

A statemachine class designed for a cycle-based approach is provided to
allow modeling complex device behaviour in an event-driven fashion.

A device may initialize a statemachine on construction, telling it what
states the device can be in and what conditions should cause it to
transition between them. The statemachine will automatically check
eligible (exiting current state) transition conditions every cycle and
perform transitions as necessary, triggering callbacks for any event
that occurs. The following events are available for every state:

-  ``on_exit`` is triggered once just before exiting the state
-  ``on_entry`` is triggered once when entering the state
-  ``in_state`` is triggered every cycle that ends in the state

Every cycle will trigger exactly one ``in_state`` event. This will
always be the last event of the cycle. When no transition occurs, this
is the only event. On the very first cycle of a simulation run,
``on_entry`` is raised against the initial state before raising an
``in_state`` against it. Any other cycles that involve a transition
first raise ``on_exit`` against the current state, and then raise
``on_entry`` and ``in_state`` against the new state. Only one transition
may occur per cycle.

There are three ways to specify event handlers when initializing the
statemacine:

-  Object-Oriented: Implement one class per state, derived from
   ``State``, which optioinally contains up to one of each event handler
-  Function-Driven: Bind individual functions to individual events that
   need handling
-  Implicit: Implement handlers in the device class, with standard names
   like ``on_entry_init`` for a state called "init", and call
   ``bindHandlersByName()``

Usage with Docker
-----------------

Docker Engine must be installed in order to run the Plankton Docker
image. Detailed installation instructions for various OSes may be found
`here <https://docs.docker.com/engine/installation/>`__.

On OSX and Windows, we recommend simply installing the `Docker
Toolbox <https://www.docker.com/products/docker-toolbox>`__. It contains
everything you need and is (currently) more stable than the "Docker for
Windows/Mac" beta versions.

On Linux, to avoid manually copy-pasting your way through the rather
detailed instructions linked to above, you can let the Docker
installation script take care of everything for you:

::

    $ curl -fsSL https://get.docker.com/ | sh

Once Docker is installed, Plankton can be run using the following
general format:

::

    $ docker run -it [docker args] dmscid/plankton [plankton args] [-- [adapter args]]

For example, to simulate a Linkam T95 **d**\ evice and expose it via the
TCP Stream **p**\ rotocol:

::

    $ docker run -it dmscid/plankton -d linkam_t95 -p stream

Details about parameters for the various adapters, and differences
between OSes are covered in the "Adapter Specifics" sections.

Usage with Python
-----------------

To use Plankton directly via Python you must install its dependencies:

-  Python 2.7+ or 3.4+
-  EPICS Base R3.14.12.5
-  PIP 8.1+

Clone the repository in a location of your choice:

::

    $ git clone https://github.com/DMSC-Instrument-Data/plankton.git

If you do not have `git <https://git-scm.com/>`__ available, you can
also download this repository as an archive and unpack it somewhere. A
few additional dependencies must be installed. This can be done through
pip via the requirements.txt file:

::

    $ pip install -r requirements.txt

If you also want to run Plankton's unit tests, you may also install the
development dependencies:

::

    $ pip install -r requirements-dev.txt

If you want to use the EPICS adapter, you will also need to configure
EPICS environment variables correctly. If you only want to communicate
using EPICS locally via the loopback device, you can configure it like
this:

::

    $ export EPICS_CA_AUTO_ADDR_LIST=NO
    $ export EPICS_CA_ADDR_LIST=localhost
    $ export EPICS_CAS_INTF_ADDR_LIST=localhost

Once all dependencies and requirements are satisfied, Plankon can be run
using the following general format (from inside the Plankton directory):

::

    $ python simulation.py [plankton args] [-- [adapter args]]

You can then run Plankton as follows (from within the plankton
directory):

::

    $ python simulation.py -d chopper -p epics

Details about parameters for the various adapters, and differences
between OSes are covered in the "Adapter Specifics" sections.

EPICS Adapter Specifics
-----------------------

The EPICS adapter takes only one optional argument:

-  ``-p`` / ``--prefix``: This string is prefixed to all PV names.
   Defaults to empty / no prefix.

Arguments meant for the adapter should be separated from general
Plankton arguments by a free-standing ``--``. For example:

::

    $ docker run -itd dmscid/plankton -d chopper -p epics -- -p SIM1:
    $ python simulation.py -d chopper -p epics -- --prefix SIM2:

When using the EPICS adapter within a docker container, the PV will be
served on the docker0 network (172.17.0.0/16).

On Linux, this means that ``EPICS_CA_ADDR_LIST`` must include this
networks broadcast address:

::

    $ export EPICS_CA_AUTO_ADDR_LIST=NO
    $ export EPICS_CA_ADDR_LIST=172.17.255.255
    $ export EPICS_CAS_INTF_ADDR_LIST=localhost

On Windows and OSX, the docker0 network is inside of a virtual machine.
To communicate with it, an EPICS Gateway to forward EPICS requests and
responses is required. We provide an `EPICS Gateway Docker
image <https://hub.docker.com/r/dmscid/epics-gateway/>`__ that can be
used to do this relatively easily. Detailed instructions can be found on
the linked page.

Stream Adapter Specifics
------------------------

The TCP Stream adapter has the following optional arguments:

-  ``-b`` / ``--bind-address``: Address of network adapter to listen on.
   Defaults to "0.0.0.0" (all network adapters).
-  ``-p`` / ``--port``: Port to listen for connections on. Defaults to
   9999.

Arguments meant for the adapter should be separated from general
Plankton arguments by a free-standing ``--``. For example:

::

    $ docker run -itd dmscid/plankton -d linkam_t95 -p stream -- -p 1234
    $ python simulation.py -d linkam_t95 -p stream -- --bind-address localhost

When using Plankton via Docker on Windows and OSX, the container will be
running inside a virtual machine, and so the port it is listening on
will be on a network inside the VM. To connect to it from outside of the
VM, an additional argument must be passed to Docker to forward the port:

::

    $ docker run -it -p 1234:4321 dmscid/plankton -d linkam_t95 -p stream -- -p 4321
    $ telnet 192.168.99.100 1234

This ``-p`` argument links port 4321 on the container to port 1234 on
the VM network adapter. It must appear after ``docker run`` and before
``dmscid/plankton``. This allows us to connect to the container from
outside of the VM, in this case using Telnet. The ``192.168.99.100`` IP
is the IP of the VM on the bridge network between the host and the VM.
VirtualBox will typically use this IP when available, but it may be
different on your system.

.. |Build Status| image:: https://travis-ci.org/DMSC-Instrument-Data/plankton.svg?branch=master
   :target: https://travis-ci.org/DMSC-Instrument-Data/plankton
.. |Coverage Status| image:: https://coveralls.io/repos/github/DMSC-Instrument-Data/plankton/badge.svg?branch=master
   :target: https://coveralls.io/github/DMSC-Instrument-Data/plankton?branch=master
