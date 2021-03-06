.. _faq:

================================
Frequently Asked Questions (FAQ)
================================


AttributeError: 'module' object has no attribute 'PiCamera'
===========================================================

You've named your script ``picamera.py`` (or you've named some other script
``picamera.py``. If you name a script after a system or third-party package you
will break imports for that system or third-party package. Delete or rename
that script (and any associated ``.pyc`` files), and try again.

Can I put the preview in a window?
==================================

No. The camera module's preview system is quite crude: it simply tells the GPU
to overlay the preview on the Pi's video output. The preview has no knowledge
(or interaction with) the X-Windows environment (incidentally, this is why the
preview works quite happily from the command line, even without anyone logged
in).

That said, the preview area can be resized and repositioned via the
:attr:`~picamera.renderers.PiRenderer.window` attribute of the
:attr:`~picamera.camera.PiCamera.preview` object. If your program can respond
to window repositioning and sizing events you can "cheat" and position the
preview within the borders of the target window. However, there's currently no
way to allow anything to appear on top of the preview so this is an imperfect
solution at best.

Help! I started a preview and can't see my console!
===================================================

As mentioned above, the preview is simply an overlay over the Pi's video
output.  If you start a preview you may therefore discover you can't see your
console anymore and there's no obvious way of getting it back. If you're
confident in your typing skills you can try calling
:meth:`~picamera.camera.PiCamera.stop_preview` by typing "blindly" into your
hidden console. However, the simplest way of getting your display back is
usually to hit ``Ctrl+D`` to terminate the Python process (which should also
shut down the camera).

When starting a preview, you may want to set the *alpha* parameter of the
:meth:`~picamera.camera.PiCamera.start_preview` method to something like 128.
This should ensure that when the preview is displayed, it is partially
transparent so you can still see your console.

The preview doesn't work on my PiTFT screen
===========================================

The camera's preview system directly overlays the Pi's output on the HDMI or
composite video ports. At this time, it will not operate with GPIO-driven
displays like the PiTFT. Some projects, like the `Adafruit Touchscreen Camera
project`_, have approximated a preview by rapidly capturing unencoded images
and displaying them on the PiTFT instead.

.. _Adafruit Touchscreen Camera project: https://learn.adafruit.com/diy-wifi-raspberry-pi-touch-cam/overview

How much power does the camera require?
=======================================

The camera `requires 250mA`_ when running. Note that simply creating a
:class:`~picamera.camera.PiCamera` object means the camera is running (due to the
hidden preview that is started to allow the auto-exposure algorithm to run). If
you are running your Pi from batteries, you should
:meth:`~picamera.camera.PiCamera.close` (or destroy) the instance when the camera is
not required in order to conserve power. For example, the following code
captures 60 images over an hour, but leaves the camera running all the time::

    import picamera
    import time

    with picamera.PiCamera() as camera:
        camera.resolution = (1280, 720)
        time.sleep(1) # Camera warm-up time
        for i, filename in enumerate(camera.capture_continuous('image{counter:02d}.jpg')):
            print('Captured %s' % filename)
            # Capture one image a minute
            time.sleep(60)
            if i == 59:
                break

By contrast, this code closes the camera between shots (but can't use the
convenient :meth:`~picamera.camera.PiCamera.capture_continuous` method as a
result)::

    import picamera
    import time

    for i in range(60):
        with picamera.PiCamera() as camera:
            camera.resolution = (1280, 720)
            time.sleep(1) # Camera warm-up time
            filename = 'image%02d.jpg' % i
            camera.capture(filename)
            print('Captured %s' % filename)
        # Capture one image a minute
        time.sleep(59)

.. note::

    Please note the timings in the scripts above are approximate. A more
    precise example of timing is given in :ref:`timelapse_capture`.

If you are experiencing lockups or reboots when the camera is active, your
power supply may be insufficient. A practical minimum is 1A for running a Pi
with an active camera module; more may be required if additional peripherals
are attached.

.. _requires 250mA: http://www.raspberrypi.org/help/faqs/#cameraPower

How can I take two consecutive pictures with equivalent settings?
=================================================================

See the :ref:`consistent_capture` recipe.

Can I use picamera with a USB webcam?
=====================================

No. The picamera library relies on libmmal which is specific to the Pi's camera
module.

How can I tell what version of picamera I have installed?
=========================================================

The picamera library relies on the setuptools package for installation
services.  You can use the setuptools ``pkg_resources`` API to query which
version of picamera is available in your Python environment like so::

    >>> from pkg_resources import require
    >>> require('picamera')
    [picamera 1.2 (/usr/local/lib/python2.7/dist-packages)]
    >>> require('picamera')[0].version
    '1.2'

If you have multiple versions installed (e.g. from ``pip`` and ``apt-get``)
they will not show up in the list returned by the ``require`` method. However,
the first entry in the list will be the version that ``import picamera`` will
import.

If you receive the error "No module named pkg_resources", you need to install
the ``pip`` utility. This can be done with the following command in Raspbian::

    $ sudo apt-get install python-pip

How come I can't upgrade to the latest version?
===============================================

If you are using Raspbian, firstly check that you haven't got both a PyPI
(``pip``) and an apt (``apt-get``) installation of picamera installed
simultaneously. If you have, one will be taking precedence and it may not be
the most up to date version.

Secondly, please understand that while the PyPI release process is entirely
automated (so as soon as a new picamera release is announced, it will be
available on PyPI), the release process for Raspbian packages is semi-manual.
There is typically a delay of a few days after a release before updated
picamera packages become accessible in the Raspbian repository.

Users desperate to try the latest version may choose to uninstall their
``apt`` based copy (uninstall instructions are provided in the
:ref:`installation instructions <raspbian_install2>`, and install using
:ref:`pip instead <system_install2>`. However, be aware that keeping a PyPI
based installation up to date is a more manual process (sticking with ``apt``
ensures everything gets upgraded with a simple ``sudo apt-get upgrade``
command).

Why is there so much latency when streaming video?
==================================================

The first thing to understand is that streaming latency is nothing to do with
the encoding or sending end of things (i.e. the Pi), but mostly to do with the
playing or receiving end. If the Pi weren't capable of encoding a frame before
the next frame arrived, it wouldn't be capable of recording video at all
(because its internal buffers would rapidly become filled with unencoded
frames).

So, why do players typically introduce several seconds worth of latency? The
primary reason is that most players (e.g. VLC) are optimized for playing
streams over a network. Such players allocate a large (multi-second) buffer and
only start playing once this is filled to guard against possible future packet
loss.

A secondary reason that all such players allocate at least a couple of frames
worth of buffering is that the MPEG standard includes certain frame types that
require it:

* I-frames (intra-frames, also known as "key frames"). These frames contain a
  complete picture and thus are the largest sort of frames. They occur at the
  start of playback and at periodic points during the stream.
* P-frames (predicted frames). These frames describe the changes from the prior
  frame to the current frame, therefore one must have successfully decoded the
  prior frame in order to decode a P-frame.
* B-frames (bi-directional predicted frames). These frames describe the changes
  from the next frame to the current frame, therefore one must have
  successfully decoded the *next* frame in order to decode the current B-frame.

B-frames aren't produced by the Pi's camera (or, as I understand it, by most
real-time recording cameras) as it would require buffering yet-to-be-recorded
frames before encoding the current one. However, most recorded media (DVDs,
BDs, and hence network video streams) do use them, so players must support
them. It is simplest to write such a player by assuming that any source may
contain B-frames, and buffering at least 2 frames worth of data at all times to
make decoding them simpler.

As for the network in between, a slow wifi network may introduce a frame's
worth of latency, but not much more than that. Check the ping time across your
network; it's likely to be less than 30ms in which case your network cannot
account for more than a frame's worth of latency.

TL;DR: the reason you've got lots of latency when streaming video is nothing to
do with the Pi. You need to persuade your video player to reduce or forgo its
buffering.

