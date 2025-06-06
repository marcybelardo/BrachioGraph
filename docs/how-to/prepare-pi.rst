.. _prepare-pi:

How to prepare a headless Raspberry Pi Zero to drive the plotter
=================================================================

A headless Raspberry Pi Zero is ideal as the engine for the plotter. It can
can be used in OTG (on-the-go) mode, in which it receives both power and a
network connection to a host machine over a single USB connection. 

Here's a recipe for a quick set-up, from scratch.

These instuctions have been tested with Raspberry Pi OS Bullseye.

Download the latest Raspberry Pi OS Lite (minimal image) from the `Raspberry Pi OS downloads page
<https://www.raspberrypi.org/downloads/raspberry-pi-os>`_.

Do whatever `needs to be done
<https://www.raspberrypi.org/documentation/installation/installing-images/>`_ to put it onto a micro SD card.


Enable SSH and OTG Ethernet access
----------------------------------

SSH access
~~~~~~~~~~

The SD card should have a ``boot`` volume. Create a file called ``ssh`` at the root.


OTG Ethernet access
~~~~~~~~~~~~~~~~~~~

"On-the-go" power/Ethernet connectivity allows you to power a Raspberry Pi Zero, and connect to it via Ethernet over
USB, on the same port (the Pi's USB port).

Edit ``config.txt``, adding::

    dtoverlay=dwc2

to a new line at the end.

Edit ``cmdline.txt``, adding::

    modules-load=dwc2,g_ether

just after ``rootwait``.

..  admonition:: Raspian Bookworm

    Raspbian Bookworm (Debian 12) Network-Manager configuration ignores RNDIS devices by default, so you will need to add this snippet to ``firstrun.txt``, immediately before the line ``rm -f /boot/firstrun.sh``::

        # Remove the rule setting gadget devices to be unmanagend
        cp /usr/lib/udev/rules.d/85-nm-unmanaged.rules /etc/udev/rules.d/85-nm-unmanaged.rules
        sed 's/^[^#]*gadget/#\ &/' -i /etc/udev/rules.d/85-nm-unmanaged.rules

        # Create a NetworkManager connection file that tries DHCP first
        CONNFILE1=/etc/NetworkManager/system-connections/usb0-dhcp.nmconnection
        UUID1=$(uuid -v4)
        cat <<- EOF >${CONNFILE1}
            [connection]
            id=usb0-dhcp
            uuid=${UUID1}
            type=ethernet
            interface-name=usb0
            autoconnect-priority=100
            autoconnect-retries=2
            [ethernet]
            [ipv4]
            dhcp-timeout=3
            method=auto
            [ipv6]
            addr-gen-mode=default
            method=auto
            [proxy]
            EOF

        # Create a NetworkManager connection file that assigns a Link-Local address if DHCP fails
        CONNFILE2=/etc/NetworkManager/system-connections/usb0-ll.nmconnection
        UUID2=$(uuid -v4)
        cat <<- EOF >${CONNFILE2}
            [connection]
            id=usb0-ll
            uuid=${UUID2}
            type=ethernet
            interface-name=usb0
            autoconnect-priority=50
            [ethernet]
            [ipv4]
            method=link-local
            [ipv6]
            addr-gen-mode=default
            method=auto
            [proxy]
            EOF

        # NetworkManager will ignore nmconnection files with incorrect permissions so change them here
        chmod 600 ${CONNFILE1}
        chmod 600 ${CONNFILE2}



Eject the card and put it into the Pi.


Connect to the Pi via OTG USB
-----------------------------

Connect a USB cable to the USB port (marked *USB*, not to be confused with the *PWR* port next to it) from your own
computer. This will provide power *and* establish an Ethernet connection to the Pi.

After a while, your machine's networking configuration should show the Raspberry Pi.

Macintosh: the Pi will appear as ``RNDIS/Ethernet Gadget`` (you can rename this).

Ubuntu: the Pi will show up as an ethernet device named ``Wired connection #``

You should be able to SSH into it::

    ssh pi@raspberrypi.local

The password is ``raspberry``.

But better than using a password is to...


Set up SSH key authentication to the Pi
---------------------------------------

Copy your public key to the Pi so you don't have to log in each time you SSH - on your computer, run::

    ssh-copy-id pi@raspberrypi.local


Set a fixed MAC address
-----------------------

You may find that by default, the Pi will generate a new MAC address and appear as a new device to the host each time
it reboots, which is annoying.

To set a fixed address, add a file ``/etc/modprobe.d/rndis.conf``. In it, add::

    options g_ether host_addr=ae:ad:f5:9d:9f:ba dev_addr=7a:26:9f:3e:97:6c

See `How can I make a Pi Zero appear as the same RNDIS/Ethernet Gadget device to the host OS each time it restarts?
<https://raspberrypi.stackexchange.com/a/104749/42583>`_ on StackExchange for more information.


Share your Internet connection to the Pi
----------------------------------------

Macintosh: this is available via the Sharing Preference Pane.

Ubuntu: go to the `IPv4 Settings` networking configuration tab, and set the method to `Shared to other computers`.

Check that you can ping an external site from the Pi.


Update everything
-----------------

Run::

    sudo apt-get update
    sudo apt-get -y upgrade

to update the software.

This will take a while.


Install the software
-------------------------------

Refer to the :ref:`install-software` section if you need more information about what will be installed by the following commands.


System packages
~~~~~~~~~~~~~~~

Run::

    sudo apt install -y python3-venv pigpiod libjbig0 libjpeg-dev liblcms2-2 libopenjp2-7 libtiff5 libwebp6 libwebpdemux2 libwebpmux3 libzstd1 libatlas3-base libgfortran5 git tmux

tmux
^^^^

This also installs `tmux <https://thoughtbot.com/blog/a-tmux-crash-course>`_, a very handy way of managing terminal
sessions, so that even if your connection is broken, you can re-join the session without losing your place - when working with a Raspberry Pi Zero connected using OTG Ethernet, this is a great convenience.


Download the BrachioGraph library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    git clone git@github.com:evildmp/BrachioGraph.git  # if you have provided GitHub with your public key
    git clone https://github.com/evildmp/BrachioGraph.git  # if you have not


Python packages
~~~~~~~~~~~~~~~

In a :ref:`Python 3 virtual environment <set-up-venv>`::

    pip install -r BrachioGraph/requirements.txt


Add a pin header
----------------

If you don't already have them, you will need a GPIO (general-purpose input/output) pin header
to connect the Raspberry Pi to the jumper wires that will connect to the servo motors.
Different pin headers are available that can be snapped or soldered into place.


Start it all up
---------------

::

    sudo pigpiod && source env/bin/activate && cd BrachioGraph && python
