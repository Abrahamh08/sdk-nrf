.. _nrf53_audio_app:

nRF5340 Audio
#############

.. contents::
   :local:
   :depth: 2

The nRF5340 Audio application demonstrates audio playback over isochronous channels (ISO) using LC3 codec compression and decompression, as per `Bluetooth® LE Audio specifications`_.
It is developed for use with the :ref:`nrf53_audio_app_dk`.

In its default configuration, the application requires access to the external repository containing the LC3 software codec.
The application also comes with various tools, including the :file:`buildprog.py` Python script that simplifies building and programming the firmware.

.. _nrf53_audio_app_overview:

Overview
********

The application can work as a gateway or a headset.
The gateway receives the audio data from external sources (USB or I2S) and forwards it to one or more headsets.
The headset is a receiver device that plays back the audio it gets from the gateway.

Both device types use the same code base, but different firmware, and you need both types of devices for testing the application.
Gateways and headsets can both run in one of the available application modes, either the *connected isochronous stream* (CIS) mode or in the *broadcast isochronous stream* (BIS) mode.
The CIS mode is the default mode of the application.

Changing configuration related to the device type and the application modes requires rebuilding the firmware and reprogramming the development kits.

Regardless of the configuration, the application handles the audio data in the following manner:

1. The gateway receives audio data from the audio source over USB or I2S.
#. The gateway processes the audio data in its application core, which channels the data through the application layers:

   a. Audio data is sent to the synchronization module (I2S-based firmware) or directly to the software codec (USB-based firmware).
   #. Audio data is encoded by the software codec.
   #. Encoded audio data is sent to the Bluetooth LE Host.

#. The host sends the encoded audio data to the LE Audio Controller Subsystem for nRF53 on the network core.
#. The subsystem forwards the audio data to the hardware radio and sends it to the headset devices, as per the LE Audio specifications.
#. The headsets receive the encoded audio data on their hardware radio on the network core side.
#. The LE Audio Controller Subsystem for nRF53 running on each of the headsets sends the encoded audio data to the Bluetooth LE Host on the headsets' application core.
#. The headsets process the audio data in their application cores, which channel the data through the application layers:

   a. Audio data is sent to the stream control module and placed in a FIFO buffer.
   #. Audio data is sent from the FIFO buffer to the synchronization module (headsets only use I2S-based firmware).
   #. Audio data is decoded by the software codec.

#. Decoded audio data is sent to the hardware audio output over I2S.

.. note::
   Currently, only a unidirectional stream is supported (gateway to headsets).
   In addition, only the gateway uses USB.
   This means that no decoded audio data is sent over USB in the current version.

In the `I2S-based firmware for gateway and headsets`_, sending the audio data through the application layers includes a mandatory synchronization step using the synchronization module.
This proprietary module ensures that the audio is played at the same time with the correct speed.
For more information, see `Synchronization module overview`_.

.. _nrf53_audio_app_overview_modes:

Application modes
=================

The application can work either in the *connected isochronous stream* (CIS) mode or in the *broadcast isochronous stream* (BIS) mode, depending on the chosen firmware configuration.

.. figure:: /images/octave_application_topologies.svg
   :alt: CIS and BIS mode overview

   CIS and BIS mode overview

Connected Isochronous Stream (CIS)
  CIS is a bidirectional communication protocol that allows for sending separate connected audio streams from a source device to one or more receivers.
  The gateway can send the audio data using both the left and the right ISO channels at the same time, allowing for stereophonic sound reproduction with synchronized playback.

  This is the default configuration of the nRF5340 Audio application.
  In this configuration, you can use the nRF5340 Audio development kit in the role of the gateway, the left headset, or the right headset.

  .. note::
     In the current version of the nRF5340 Audio application, the CIS mode offers only monodirectional communication.

Broadcast Isochronous Stream (BIS)
  BIS is a monodirectional communication protocol that allows for broadcasting one or more audio streams from a source device to an unlimited number of receivers that are not connected to the source.

  In this configuration, you can use the nRF5340 Audio development kit in the role of the gateway or as one of the headsets.
  Use multiple nRF5340 Audio development kits to test BIS having multiple receiving headsets.

  .. note::
     * In the BIS mode, you can use any number of nRF5340 Audio development kits as receivers.
     * In the current version of the nRF5340 Audio application, the BIS mode offers only monophonic sound reproduction.

The audio quality for both modes does not change, although the processing time for stereo can be longer.

.. _nrf53_audio_app_overview_architecture:

Firmware architecture
=====================

The following figure illustrates the software layout for the nRF5340 Audio application:

.. figure:: /images/octave_application_structure_generic.svg
   :alt: nRF5340 Audio high-level design (overview)

   nRF5340 Audio high-level design (overview)

The network core of the nRF5340 SoC runs the *LE Audio Controller Subsystem for nRF53*.
This subsystem is a Bluetooth LE Controller that is custom-made for the application.
It is responsible for receiving the audio stream data from hardware layers and forwarding the data to the Bluetooth LE host on the application core.
The subsystem implements the lower layers of the Bluetooth Low Energy software stack and follows the LE Audio specification requirements.

The application core runs both the Bluetooth LE Host from Zephyr and the application layer.
The application layer is composed of a series of modules from different sources.
These modules include the following major ones:

* Peripheral modules from the |NCS|:

  * I2S
  * USB
  * SPI
  * TWI/I2C
  * UART (debug)
  * Timer

* Application-specific Bluetooth modules for handling the Bluetooth connection:

  * :file:`le_audio_cis_gateway.c` or :file:`le_audio_cis_headset.c` - One of these ``cis`` modules is used by default.
  * :file:`le_audio_bis_gateway.c` or :file:`le_audio_bis_headset.c` - One of these ``bis`` modules is selected automatically when you :ref:`switch to the BIS configuration <nrf53_audio_app_configuration_select_bis>`.

  Only one of these files is used at compile time.
  Each of these files handles the Bluetooth connection and Bluetooth events and funnels the data to the relevant audio modules.

* Application-specific custom modules:

  * Stream Control - This module implements a simple state machine for the application (``STREAMING`` or ``PAUSED``).
    It also handles events from Bluetooth LE and buttons, receives audio from the host, and forwards the audio data to the next module.
  * FIFO buffers
  * Synchronization module (part of `I2S-based firmware for gateway and headsets`_) - See `Synchronization module overview`_ for more information.

* Application-specific modules from external sources:

  * LC3 encoder/decoder (default)

.. note::
   :ref:`Selecting and configuring the right software codec <nrf53_audio_app_requirements_codec>` is required to run the application.

Since the application architecture is uniform and the firmware code is shared, the set of audio modules in use depends on the chosen stream mode (BIS or CIS), the chosen audio inputs and outputs (USB or analog jack), and if the gateway or the headset configuration is selected.

.. note::
   In the current version of the application, no bootloader is used, and device firmware update (DFU) is not supported.

.. _nrf53_audio_app_overview_architecture_usb:

USB-based firmware for gateway
------------------------------

The following figure shows an overview of the modules currently included in the firmware that uses USB:

.. figure:: /images/octave_application_structure_gateway.svg
   :alt: nRF5340 Audio modules on the gateway using USB

   nRF5340 Audio modules on the gateway using USB

In this firmware design, no synchronization module is used after decoding the incoming frames or before encoding the outgoing ones.
The Bluetooth LE RX FIFO is mainly used to make decoding run in a separate thread.

.. _nrf53_audio_app_overview_architecture_i2s:

I2S-based firmware for gateway and headsets
-------------------------------------------

The following figure shows an overview of the modules currently included in the firmware that uses I2S:

.. figure:: /images/octave_application_structure.svg
   :alt: nRF5340 Audio modules on the gateway and the headsets using I2S

   nRF5340 Audio modules on the gateway and the headsets using I2S

The Bluetooth LE RX FIFO is mainly used to make :file:`audio_datapath.c` (synchronization module) run in a separate thread.
After encoding the audio data received from I2S, the frames are sent by the encoder thread using a function located in :file:`streamctrl.c`.

.. _nrf53_audio_app_overview_architecture_sync_module:

Synchronization module overview
-------------------------------

The synchronization module (:file:`audio_datapath.c`) handles audio synchronization.
To synchronize the audio, it executes the following types of adjustments:

* Presentation compensation
* Drift compensation

The presentation compensation makes all the headsets play audio at the same time, even if the packets containing the audio frames are not received at the same time on the different headsets.
In practice, it moves the audio data blocks in the FIFO forward or backward a few blocks, adding blocks of *silence* when needed.

The drift compensation adjusts the frequency of the audio clock to adjust the speed at which the audio is played.
This is required in the CIS mode, where the gateway and headsets must keep the audio playback synchronized to provide True Wireless Stereo (TWS) audio playback.
As such, it provides both larger adjustments at the start and then continuous small adjustments to the audio synchronization.
This compensation method counters any drift caused by the differences in the frequencies of the quartz crystal oscillators used in the development kits.
Development kits use quartz crystal oscillators to generate a stable clock frequency.
However, the frequency of these crystals always slightly differs.
The drift compensation makes the inter-IC sound (I2S) interface on the headsets run as fast as the Bluetooth packets reception.
This prevents I2S overruns or underruns, both in the CIS mode and the BIS mode.

See the following figure for an overview of the synchronization module.

.. figure:: /images/octave_application_structure_sync_module.svg
   :alt: nRF5340 Audio synchronization module overview

   nRF5340 Audio synchronization module overview

Both synchronization methods use the SDU reference timestamps (:c:type:`sdu_ref`) as the reference variable.
If the device is a gateway that is :ref:`using I2S as audio source <nrf53_audio_app_overview_architecture_i2s>` and the stream is unidirectional (gateway to headsets), :c:type:`sdu_ref` is continuously being extracted from the LE Audio Controller Subsystem for nRF53 on the gateway.
The extraction happens inside the :file:`le_audio_cis_gateway.c` and :file:`le_audio_bis_gateway.c` files' send function.
The :c:type:`sdu_ref` values are then sent to the gateway's synchronization module, and used to do drift compensation.

.. note::
   Inside the synchronization module (:file:`audio_datapath.c`), all time-related variables end with ``_us`` (for microseconds).
   This means that :c:type:`sdu_ref` becomes :c:type:`sdu_ref_us` inside the module.

As the nRF5340 is a dual-core SoC, and both cores need the same concept of time, each core runs a free-running timer in an infinite loop.
These two timers are reset at the same time, and they run from the same clock source.
This means that they should always show the same values for the same points in time.
The network core of the nRF5340 running the LE controller for nRF53 uses its timer to generate the :c:type:`sdu_ref` timestamp for every audio packet received.
The application core running the nRF5340 Audio application uses its timer to generate :c:type:`cur_time` and :c:type:`frame_start_ts`.

After the decoding takes place, the audio data is divided into smaller blocks and added to a FIFO.
These blocks are then continuously being fed to I2S, block by block.

See the following figure for the details of the compensation methods of the synchronization module.

.. figure:: /images/octave_application_sync_module_states.svg
   :alt: nRF5340 Audio's state machine for compensation mechanisms

   nRF5340 Audio's state machine for compensation mechanisms

The following external factors can affect the presentation compensation:

* The drift compensation must be synchronized to the locked state (:c:enumerator:`DRIFT_STATE_LOCKED`) before the presentation compensation can start.
  This drift compensation adjusts the frequency of the audio clock, indicating that the audio is being played at the right speed.
  When the drift compensation is not in the locked state, the presentation compensation does not leave the init state (:c:enumerator:`PRES_STATE_INIT`).
  Also, if the drift compensation loses synchronization, moving out of :c:enumerator:`DRIFT_STATE_LOCKED`, the presentation compensation moves back to :c:enumerator:`PRES_STATE_INIT`.
* When audio is being played, it is expected that a new audio frame is received in each ISO connection interval.
  If this does not occur, the headset might have lost its connection with the gateway.
  When the connection is restored, the application receives an :c:type:`sdu_ref` not consecutive with the previously received :c:type:`sdu_ref`.
  Then the presentation compensation is put into :c:enumerator:`PRES_STATE_WAIT` to ensure that the audio is still in sync.

.. note::
   When both the drift and presentation compensation are in state *locked* (:c:enumerator:`DRIFT_STATE_LOCKED` and :c:enumerator:`PRES_STATE_LOCKED`), **LED2** lights up.

Synchronization module flow
+++++++++++++++++++++++++++

The received audio data in the I2S-based firmware devices follows the following path:

1. The LE Audio Controller Subsystem for nRF53 running on the network core receives the compressed audio data.
#. The controller subsystem sends the audio data to the Zephyr Bluetooth LE host similarly to the :ref:`zephyr:bluetooth-hci-rpmsg-sample` sample.
#. The host sends the data to the stream control module (:file:`streamctrl.c`).
#. The data is sent to a FIFO buffer.
#. The data is sent from the FIFO buffer to the :file:`audio_datapath.c` synchronization module.
   The :file:`audio_datapath.c` module performs the audio synchronization based on the SDU reference timestamps.
   Each package sent from the gateway gets a unique SDU reference timestamp.
   These timestamps are generated on the headset controllers (in the network core).
   This enables the creation of True Wireless Stereo (TWS) earbuds where the audio is synchronized in the CIS mode.
   It does also keep the speed of the inter-IC sound (I2S) interface synchronized with the sending and receiving speed of Bluetooth packets.
#. The :file:`audio_datapath.c` module sends the compressed audio data to the LC3 audio decoder for decoding.

#. The audio decoder decodes the data and sends the uncompressed audio data (PCM) back to the :file:`audio_datapath.c` module.
#. The :file:`audio_datapath.c` module continuously feeds the uncompressed audio data to the hardware codec.
#. The hardware codec receives the uncompressed audio data over the inter-IC sound (I2S) interface and performs the digital-to-analog (DAC) conversion to an analog audio signal.

.. _nrf53_audio_app_requirements:

Requirements
************

The nRF5340 Audio application is designed to be used only with the following hardware:

+---------------------+----------------------------------+--------------------------+---------------------------------+
| Hardware platforms  | PCA                              | Board name               | Build target                    |
+=====================+==================================+==========================+=================================+
| nRF5340 Audio DK    | PCA10121 revision 1.0.0 or above | nrf5340_audio_dk_nrf5340 | nrf5340_audio_dk_nrf5340_cpuapp |
+---------------------+----------------------------------+--------------------------+---------------------------------+

.. note::
   The application supports PCA10121 revisions 1.0.0 or above.
   The application is also compatible with the following pre-launch revisions:

   * Revision 0.7.0 (not recommended).
   * Revisions 0.8.0 and above.

You need at least two nRF5340 Audio development kits (one with the gateway firmware and one with headset firmware) to test the application.
For CIS with TWS in mind, three kits are required.

.. _nrf53_audio_app_requirements_codec:

Software codec requirements
===========================

The nRF5340 Audio application must the LC3 software (developed specifically for use with LE Audio).
The codec requires :ref:`adding its own repository before building and running <nrf53_audio_app_configuration_repos>`.

|lc3_codec_access_note|
See :ref:`nrf53_audio_app_configuration_select_codec` for more information.

.. _nrf53_audio_app_dk:

nRF5340 Audio development kit
=============================

The nRF5340 Audio development kit is a hardware development platform that demonstrates the nRF5340 Audio application.

.. _nrf53_audio_app_dk_features:

Key features of the nRF5340 Audio DK
------------------------------------

* Nordic Semiconductor's nRF5340 Bluetooth LE / multiprotocol SoC.
* Nordic Semiconductor's nPM1100 power management SoC.
* CS47L63 AD-DA converter from Cirrus Logic, dedicated to TWS devices.
* Stereo analog line input.
* Mono analog output.
* Onboard Pulse Density Modulation (PDM) microphone.
* Computer connection and battery charging through USB-C.
* Second nRF5340 SoC that works as an onboard SEGGER debugger.
* SD card reader (no SD card supplied).
* User-programmable buttons and LEDs.
* Normal operating temperature range 10–40°C.

  .. note::
      The battery supplied with this kit can operate with a max temperature of Max +60°C.

* When using a power adapter to USB, the power supply adapter must meet USB power supply requirements.
* Embedded battery charge system.
* Rechargeable Li-Po battery with 1500 mAh capacity.

.. _nrf53_audio_app_dk_drawings:

Hardware drawings
-----------------

The nRF5340 Audio hardware drawings show both sides of the development kit in its plastic case:

.. figure:: /images/nRF5340_audio_dk_front_case.svg
   :alt: Figure 1. nRF5340 Audio DK (PCA10121) front view

   Figure 1. nRF5340 Audio DK (PCA10121) front view

.. figure:: /images/nRF5340_audio_dk_back_case.svg
   :alt: Figure 2. nRF5340 Audio DK (PCA10121) back view

   Figure 2. nRF5340 Audio DK (PCA10121) back view

The following figure shows the back of the development kit without the case:

.. figure:: /images/nRF5340_audio_dk_back.svg
   :alt: Figure 3. nRF5340 Audio DK (PCA10121) back view without case

   Figure 3. nRF5340 Audio DK (PCA10121) back view without case

For the description of the relevant PCB elements, see the `User interface`_ section.

.. _nrf53_audio_app_dk_solder_bridge_overview:

Solder bridge overview
----------------------

The nRF5340 Audio DK has a range of solder bridges for enabling or disabling selected functionalities.
Changes to these are not needed for normal use of the DK.
The following table is a complete overview of the solder bridges on the nRF5340 Audio DK.

+------------+-------------------------------------------------------------------------------------+--------------+--------+
|Designator  | Description                                                                         | Default state| Layer  |
+============+=====================================================================================+==============+========+
|SB1         | Short to connect digital microphone DOUT to P1.06                                   | Open         | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB2         | Cut to disconnect P0.12 from TRACE                                                  | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB3         | Short to connect PMIC MODE to VOUTB, must not be shorted while SB4 is shorted       | Open         | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB4         | Cut to disable PMIC MODE from GND, must not be shorted while SB3 is shorted         | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB5         | Cut to enable VBAT current measurements on P6                                       | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB6         | Cut to enable VBAT current measurements on P6                                       | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB7         | Cut to enable HW CODEC 1.8V current measurements on P8                              | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB8         | Cut to enable VDD_nRF current measurements on P9                                    | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB9         | Cut to disconnect filter from OUTP                                                  | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB10        | Cut to disconnect filter from OUTN                                                  | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB11        | Cut to disconnect the LED for the HW CODEC GPIO                                     | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB12        | Cut to disconnect digital microphone POWER from the HW CODEC                        | Shorted      | Bottom |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB13        | Cut to disconnect digital microphone DATA from the HW CODEC                         | Shorted      | Bottom |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB14        | Cut to disconnect digital microphone CLOCK from the HW CODEC                        | Shorted      | Bottom |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB15        | Short to connect AUX I2S MCLK to HW CODEC MCLK1                                     | Open         | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB16        | Short to connect AUX I2S MCLK to HW CODEC MCLK2                                     | Open         | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB17        | Short to connect P5 pin 6 to GND	                                                   | Open         | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB18        | Cut to disconnect P5 pin 6 from SHIELD DETECT                                       | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB19        | Cut to disconnect RTS and CTS flow control lines on UART1                           | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB20        | Cut to disconnect RTS and CTS flow control lines on UART2                           | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB21        | Cut to disconnect nRF53 RESET from RESET button when debug is disabled              | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB22        | Short to permanently connect RESET button to nRF53 RESET                            | Open         | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB23        | Cut to disconnect RESET button from interface MCU                                   | Shorted      | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+
|SB24        | Short to bypass analog switch for MCLK                                              | Open         | Top    |
+------------+-------------------------------------------------------------------------------------+--------------+--------+


.. _nrf53_audio_app_dk_testpoint_overview:

Testpoint overview
------------------

The following table is a complete overview of the test points on the nRF5340 Audio DK.

+-------------+----------------------------+--------------------------------------------------+-------+--------+
| Designator  | Net                        | Description                                      | Size  | Layer  |
+=============+============================+==================================================+=======+========+
|TP1          | NetTP1-1                   | IN1LP_1 pin of CS47L63                           | 1.5mm | Bottom |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP2          | NetTP2-1                   | IN1LN_1 pin of CS47L63                           | 1.5mm | Bottom |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP3          | NetTP3-1                   | IN1RP pin of CS47L63                             | 1.5mm | Bottom |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP4          | NetTP4-1                   | IN1RN pin of CS47L63                             | 1.5mm | Bottom |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP5          | NetTP5-1                   | IN2LN pin of CS47L63                             | 1.5mm | Bottom |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP6          | NetTP6-1                   | IN2RN pin of CS47L63                             | 1.5mm | Bottom |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP7          | HW_CODEC_AUX_I2C.SCL       | AUX SCL pin of CS47L63                           | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP8          | HW_CODEC_AUX_I2C.SDA       | AUX SDA pin of CS47L63                           | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP9          | P0.07/AIN3                 | RGB LED 1 Red color input pin                    | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP10         | P0.28/AIN7                 | RGB LED 2 Red color input pin                    | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP11         | P1.01                      | LED 3 input pin                                  | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP12         | P0.04/AIN0                 | Button 3                                         | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP13         | VDD_EXT_HW_CODEC.1V2       | External HW CODEC 1.2V supply                    | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP14         | VDD_EXT_HW_CODEC.1V8       | External HW CODEC 1.8V supply                    | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP15         | BAT_NTC                    | Li-poly battery NTC pin                          | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP16         | BATTERY                    | Li-poly battery voltage after power switch       | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP17         | NetC41-1                   | USB voltage after power switch                   | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP18         | NetC43-2                   | USB voltage before power switch                  | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP19         | HEADPHONE.OUTP             | Headphone jack tip                               | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP20         | HEADPHONE.OUTN             | Headphone jack sleeve                            | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP21         | DU_N                       | USB connector D-                                 | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP22         | DU_P                       | USB connector D+                                 | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP23         | SWDIO                      | nRF5340 Serial Wire Debug data                   | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP24         | SWDCLK                     | nRF5340 Serial Wire Debug clock                  | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP25         | R\E\S\E\T\                 | nRF5340 Reset                                    | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP26         | SD_CS                      | SD card slot CS line                             | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP27         | SD_SCK                     | SD card slot SCK line                            | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP28         | VDD_IN_1V                  | 1.2V regulator output                            | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP29         | SUPPLY_1V8                 | nPM1100 1.8V output                              | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP30         | SUPPLY_3V3                 | 3.3V regulator output                            | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP31         | VDD_DBG_3V3                | Debug regulator 3.3V output                      | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP32         | VDD_DBG_1V8                | Debug regulator 1.8V output                      | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP33         | SW_EN                      | Load switch enable signal                        | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP34         | GND                        | Ground                                           | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP35         | GND                        | Ground                                           | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP36         | NetQ9-1                    | Debug enable signal                              | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP37         | IMCU_SWDIO                 | Interface MCU Serial Wire Debug data             | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP38         | IMCU_RESET                 | Interface MCU Reset                              | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP39         | IMCU_SWDCLK                | Interface MCU Serial Wire Debug clock            | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP40         | SHIELD_DETECT              | Detect signal for Arduino compatible shield      | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP41         | HW_CODEC_IF.SPI.MISO       | SPI MISO pin of CS47L63                          | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP42         | HW_CODEC_IF.SPI.MOSI       | SPI MOSI pin of CS47L63                          | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP43         | HW_CODEC_IF.SPI.SCK        | SPI SCK pin of CS47L63                           | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP44         | HW_CODEC_IF.SPI.CS         | SPI SS pin of CS47L63                            | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP45         | HW_CODEC_IF.CTRL.GPIO      | GPIO pin of CS47L63                              | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP46         | HW_CODEC_IF.CTRL.IRQ       | IRQ pin of CS47L63                               | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP47         | HW_CODEC_IF.CTRL.RESET     | RESET pin of CS47L63                             | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP48         | HW_CODEC_IF.I2S.MCLK       | MCLK1 pin of CS47L63                             | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP49         | HW_CODEC_IF.I2S.DOUT       | I2S DOUT pin of CS47L63                          | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP50         | HW_CODEC_IF.I2S.DIN        | I2S DIN pin of CS47L63                           | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP51         | HW_CODEC_IF.I2S.BCLK       | I2S BCLK pin of CS47L63                          | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP52         | HW_CODEC_IF.I2S.FSYNC      | I2S FSYNC pin of CS47L63                         | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP53         | NetSB12-1                  | MICBIASB pin of CS47L63                          | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP54         | NetSB13-1                  | IN1_PDMDATA pin of CS47L63                       | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP55         | NetSB14-1                  | IN1_PDMCLK pin of CS47L6                         | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP56         | PMIC_ERR                   | nPM1100 error indication                         | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP57         | PMIC_CHG                   | nPM1100 charge indication                        | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP58         | P0.29                      | RGB LED 2 Green color input pin                  | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP59         | P0.30                      | RGB LED 2 Blue color input pin                   | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP60         | P1.04                      | UART1 RXD                                        | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP61         | P1.05                      | UART1 TXD                                        | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP62         | P1.06                      | UART1 CTS                                        | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP63         | P1.07                      | UART1 RTS                                        | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP64         | NetJ5-10                   | SD card slot card detect                         | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP65         | P0.11                      | SD card slot level translator enable             | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP66         | P1.15                      | Current shunt monitor alert signal               | 1.0mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP67         | GND                        | Ground                                           | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP68         | LINE_IN.LEFT               | Line-in jack tip                                 | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+
|TP69         | LINE_IN.RIGHT              | Line-in jack ring                                | 1.5mm | Top    |
+-------------+----------------------------+--------------------------------------------------+-------+--------+


.. _nrf53_audio_hw_limitations:

nRF5340 Audio hardware limitations
----------------------------------

The following table lists hardware limitations discovered in different revisions of the nRF5340 Audio DK.

.. list-table::
    :widths: auto
    :header-rows: 1

    * - PCA10121 revision
      - Limitation
      - Description
      - Workaround
      - Fixed in revision
    * - Rev 1.0.0
      - CS47L63 AD-DA converter (**U2**) may fail to start
      - In some occasions, the 1.2 V power supply for **U2** is not provided at boot-up.
        This is caused by higher than expected inrush current.
        This function is tested in production.
        The issue should not happen, although we observe that some kits have the problem.
      - Restart kit or attach the battery to the kit before connecting the USB cable.
        If problem persists, contact Nordic Semiconductor and ask for replacement.
      - Rev 1.0.1

.. _nrf53_audio_app_configuration_files:

nRF5340 Audio configuration files
=================================

The nRF5340 Audio application uses :file:`Kconfig.defaults` files to change configuration defaults automatically, based on the different application versions and device types.

Only one of the following :file:`.conf` files is included when building:

* :file:`prj.conf` is the default configuration file and it implements the debug application version.
* :file:`prj_release.conf` is the optional configuration file and it implements the release application version.
  No debug features are enabled in the release application version.
  When building using the command line, you must explicitly specify if :file:`prj_release.conf` is going to be included instead of :file:`prj.conf`.
  See :ref:`nrf53_audio_app_building` for details.

.. _nrf53_audio_app_ui:

User interface
**************

The application implements a simple user interface based on the available PCB elements.
You can control the application using predefined switches and buttons while the LEDs display information.

.. _nrf53_audio_app_ui_switches:

Switches
========

The application uses the following switches on the supported development kit:

+-------------------+-------------------------------------------------------------------------------------+
| Switch            | Function                                                                            |
+===================+=====================================================================================+
| **POWER**         | Turns the development kit on or off.                                                |
+-------------------+-------------------------------------------------------------------------------------+
| **DEBUG ENABLE**  | Turns on or off power for debug features.                                           |
|                   | This switch is used for accurate power and current measurements.                    |
+-------------------+-------------------------------------------------------------------------------------+

.. _nrf53_audio_app_ui_buttons:

Buttons
=======

The application uses the following buttons on the supported development kit:

+---------------+----------------------------------------------------------------+
| Button        | Function                                                       |
+===============+================================================================+
| **VOL-**      | Turns the playback volume down (and unmutes).                  |
+---------------+----------------------------------------------------------------+
| **VOL+**      | Turns the playback volume up (and unmutes).                    |
+---------------+----------------------------------------------------------------+
| **PLAY/PAUSE**| Starts or pauses the playback.                                 |
+---------------+----------------------------------------------------------------+
| **BTN 4**     | During playback, sends a test tone generated on the device.    |
|               | Pressing the button multiple times changes the tone frequency. |
|               | The available values are 1000 Hz, 2000 Hz, and 4000 Hz.        |
|               | Use this tone to check the synchronization of headsets.        |
|               | Sending of test tone is currently only supported on the        |
|               | gateway.                                                       |
+---------------+----------------------------------------------------------------+
| **BTN 5**     | Mutes the playback volume.                                     |
+---------------+----------------------------------------------------------------+
| **RESET**     | Resets the device.                                             |
+---------------+----------------------------------------------------------------+

.. _nrf53_audio_app_ui_leds:

LEDs
====

To indicate the tasks performed, the application uses the LED behavior described in the following table:

+--------------------------+-----------------------------------------------------------------------------------------------------+
| LED                      |Indication                                                                                           |
+==========================+=====================================================================================================+
| **LED1**                 | Off - No Bluetooth connection.                                                                      |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Blinking blue - Depending on the device and the mode:                                               |
|                          |                                                                                                     |
|                          | * Headset: Kits have started streaming audio (BIS and CIS modes).                                   |
|                          | * Gateway: Kit has connected to a headset (CIS mode) or has started broadcasting audio (BIS mode).  |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid blue - Headset, depending on the mode:                                                        |
|                          | Kits have connected to the gateway (CIS mode) or found a broadcasting stream (BIS mode).            |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **LED2**                 | Off - Sync not achieved.                                                                            |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid green - Sync achieved (both drift and presentation compensation are in the ``LOCKED`` state). |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **LED3**                 | Blinking green - The nRF5340 Audio DK application core is running.                                  |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **CODEC**                | Off - No configuration loaded to the onboard hardware codec.                                        |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid green - Hardware codec configuration loaded.                                                  |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **RGB1**                 | Solid green - The device is programmed as the gateway.                                              |
| (bottom side LEDs around +-----------------------------------------------------------------------------------------------------+
| the center opening)      | Solid blue - The device is programmed as the left headset.                                          |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid magenta - The device is programmed as the right headset.                                      |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid yellow - The device is programmed with factory firmware.                                      |
|                          | It must be re-programmed as gateway or headset.                                                     |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid red (debug mode) - Fault in the application core has occurred.                                |
|                          | See UART log for details and use the **RESET** button to reset the device.                          |
|                          | In the release mode, the device resets automatically with no indication on LED or UART.             |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **RGB 2**                | Controlled by the Bluetooth LE Controller on the network core.                                      |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Blinking green - Ongoing CPU activity.                                                              |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid red - Error.                                                                                  |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid white (all colors on) - The **RGB 2** LED is not initialized by the Bluetooth LE Controller.  |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **ERR**                  | PMIC error or a charging error (or both).                                                           |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **CHG**                  | Off - Charge completed or no battery connected.                                                     |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid yellow - Charging in progress.                                                                |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **OB/EXT**               | Off - No 3.3 V power available.                                                                     |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid green - On-board hardware codec selected.                                                     |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Solid yellow - External hardware codec selected.                                                    |
|                          | This LED turns solid yellow also when the devices are reset, for the time then pins are floating.   |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **FTDI SPI**             | Off - No data is written to the hardware codec using SPI.                                           |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Yellow - The same SPI is used for both the hardware codec and the SD card.                          |
|                          | When this LED is yellow, the shared SPI is used by the FTDI to write data to the hardware codec.    |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **IFMCU**                | Off - No PC connection available.                                                                   |
| (bottom side)            +-----------------------------------------------------------------------------------------------------+
|                          | Solid green - Connected to PC.                                                                      |
|                          +-----------------------------------------------------------------------------------------------------+
|                          | Rapid green flash - USB enumeration failed.                                                         |
+--------------------------+-----------------------------------------------------------------------------------------------------+
| **HUB**                  | Off - No PC connection available.                                                                   |
| (bottom side)            +-----------------------------------------------------------------------------------------------------+
|                          | Green - Standard USB hub operation.                                                                 |
+--------------------------+-----------------------------------------------------------------------------------------------------+

.. _nrf53_audio_app_configuration:

Configuration
*************

|config|

.. _nrf53_audio_app_configuration_repos:

Setting up the nRF5340 Audio repositories
=========================================

The application relies on the following :ref:`external OSS repositories <dm_code_base>` that need to be pulled using west:

* LC3 software codec repository
* Hardware codec driver repository

To have these repositories managed by west, complete the following steps:

1. Add the group filter specific to the nRF5340 Audio application to the west manifest file of your project by running the following command:

   .. code-block:: console

      west config manifest.group-filter +nrf5340_audio

#. Update west to fetch the repositories in the nRF5340 Audio group:

   .. code-block:: console

      west update

If west can fetch the repositories correctly, you can now build the application.

For more information about west, see :ref:`Zephyr's documentation page <zephyr:west>`.

.. _nrf53_audio_app_configuration_select_codec:

Selecting the audio software codec
==================================

The nRF5340 Audio application must use either the LC3 software (developed specifically for use with LE Audio).
The codec requires :ref:`adding its own repository before building and running <nrf53_audio_app_configuration_repos>`.

|lc3_codec_access_note|

.. _nrf53_audio_app_configuration_select_bis:

Selecting the BIS mode
======================

The CIS mode is the default operating mode for the application.
You can switch to the BIS mode by adding the ``CONFIG_TRANSPORT_BIS`` Kconfig option set to ``y``  to the :file:`prj.conf` file for the debug version and the :file:`prj_release.conf` file for the release version.

.. _nrf53_audio_app_configuration_select_i2s:

Selecting the I2S serial
========================

In the default configuration, the gateway application uses the USB serial port as the audio source.
The :ref:`nrf53_audio_app_building` and :ref:`nrf53_audio_app_testing` steps also refer to using the USB serial connection.

You can switch to using the I2S serial connection by adding the ``CONFIG_AUDIO_SOURCE_I2S`` Kconfig option set to ``y``  to the :file:`prj.conf` file for the debug version and the :file:`prj_release.conf` file for the release version.

When testing the application, an additional audio jack cable is required to use I2S.
Use this cable to connect the audio source (PC) to the analog **LINE IN** on the development kit.

.. _nrf53_audio_app_building:

Building and running
********************

This sample can be found under :file:`applications/nrf5340_audio` in the nRF Connect SDK folder structure.

.. note::
   Building and programming the nRF5340 Audio application is different from the :ref:`standard procedure <ug_nrf5340_building>` of building and programming for the nRF5340 DK.
   This is because the nRF5340 Audio application only builds and programs the files for the application core.
   |net_core_hex_note|

You can build and program the application in one of the following ways:

* :ref:`nrf53_audio_app_building_script`.
  This is the suggested method.
  Using this method allows you to build and program multiple development kits at the same time.
* :ref:`nrf53_audio_app_building_standard`.
  Using this method requires building and programming each development kit separately.

Prerequisites
=============

Before building the application, make sure to meet the following prerequisites described in the :ref:`nrf53_audio_app_configuration` section:

* :ref:`nrf53_audio_app_configuration_repos`
* :ref:`nrf53_audio_app_configuration_select_codec`

You might also want to check the :ref:`nRF5340 Audio application known issues <known_issues_nrf5340audio>`.

Testing out of the box
======================

Each development kit comes preprogrammed with basic firmware that indicates if the kit is functional.
Before building the application, you can verify if the kit is working by completing the following steps:

1. Plug the device into the USB port using USB-C.
#. Turn on the development kit using the On/Off switch.
#. Observe **RGB1** (bottom side LEDs around the center opening that illuminate the Nordic Semiconductor logo) turn solid yellow, **OB/EXT** turn solid green, and **LED3** start blinking green.

You can now program the development kits with either gateway or headset firmware before they can be used.

.. _nrf53_audio_app_building_script:

Building and programming using script
=====================================

The suggested method for building the application and programming it to the development kit is running the :file:`buildprog.py` Python script, which is located in the :file:`applications/nrf5340_audio/tools/buildprog` directory.
The script automates the process of selecting :ref:`configuration files <nrf53_audio_app_configuration_files>` and building different versions of the application.
This eases the process of building and programming images for multiple development kits.

Preparing the JSON file
-----------------------

The script depends on the settings defined in the :file:`nrf5340_audio_dk_devices.json` file.
Before using the script, make sure to update this file with the following information for each development kit you want to use:

* ``nrf5340_audio_dk_snr`` -- This field lists the SEGGER serial number.
  You can check this number on the sticker on the nRF5340 Audio development kit.
  Alternatively, connect the development kit to your PC and run ``nrfjprog -i`` in a command window to print the SEGGER serial number of the kit.
* ``nrf5340_audio_dk_dev`` -- This field assigns the specific nRF5340 Audio development kit to be a headset or a gateway.
* ``channel`` -- This field is valid only for headsets operating in the CIS mode.
  It sets the channels on which the headset is meant to work.
  When no channel is set, the headset is programmed as a left channel one.

Running the script
------------------

After editing the :file:`nrf5340_audio_dk_devices.json` file, run :file:`buildprog.py` to build the firmware for the development kits.
The building command for running the script requires providing the following parameters, in line with :ref:`nrf53_audio_app_configuration_files`:

* Core type (``-c`` parameter): ``app``, ``net``, or ``both``
* Application version (``-b`` parameter): either ``release`` or ``debug``
* Device type (``-d`` parameter): ``headset``, ``gateway``, or ``both``

For example, to build the application using the script for the application core with the ``debug`` application version for both the headset and the gateway, run the following command from the :file:`buildprog` directory:

.. code-block:: console

   python buildprog.py -c app -b debug -d both

This command can be ran from any location, as long as the correct path to :file:`buildprog.py` is given.

The build files are saved in the :file:`applications/nrf5340_audio/build` directory.
The script creates a directory for each application version and device type combination.
For example, when running the command above, the script creates the :file:`dev_gateway/build_debug` and :file:`dev_headset/build_debug` directories.

Programming with the script
   The development kits are programmed according to the serial numbers set in the JSON file.
   If you run the script with the ``-p`` parameter, you can program one or both of the cores after building the files.
   Make sure to connect the development kits with your PC using USB-C and turn them on using the **POWER** switch before you run the command.
   The command for programming can look as follows:

   .. code-block:: console

      python buildprog.py -c both -b debug -d both -p

   .. note::
      If you are using Windows Subsystem for Linux (WSL) and encounter problems while programming, include the ``-s`` parameter to program sequentially.

   This command builds the application with the ``debug`` application version for both the headset and the gateway and programs the application core.
   Given the ``-c both`` parameter, it also takes the precompiled Bluetooth Low Energy Controller binary from the :file:`applications/nrf5340_audio/bin` directory and programs it to the network core of both the gateway and the headset.

   .. note::
      If the programming command fails because of :ref:`readback_protection_error`, run :file:`buildprog.py` with the ``--recover-on-fail`` or ``-f`` parameter to recover and re-program automatically when programming fails.
      For example, using the programming command example above:

      .. code-block:: console

         python buildprog.py -c both -b debug -d both -p --recover-on-fail

Getting help
   Run ``python buildprog.py -h`` for information about all available script parameters.

Configuration table overview
   When running the script command, a table similar to the following one is displayed to provide an overview of the selected options and parameter values:

   .. code-block:: console

      +------------+----------+---------+--------------+---------------------+---------------------+
      | snr        | snr conn | device  | only reboot  | core app programmed | core net programmed |
      +------------+----------+---------+--------------+---------------------+---------------------+
      | 1010101010 | True     | headset | Not selected | Selected TBD        | Not selected        |
      | 2020202020 | True     | gateway | Not selected | Selected TBD        | Not selected        |
      | 3030303030 | True     | headset | Not selected | Selected TBD        | Not selected        |
      +------------+----------+---------+--------------+---------------------+---------------------+

   See the following table for the meaning of each column and the list of possible values:

   +-----------------------+-----------------------------------------------------------------------------------------------------+-----------------------------------------------+
   | Column                | Indication                                                                                          | Possible values                               |
   +=======================+=====================================================================================================+===============================================+
   | ``snr``               | Serial number of the device, as provided in the :file:`nrf5340_audio_dk_devices.json` file.         | Serial number.                                |
   +-----------------------+-----------------------------------------------------------------------------------------------------+-----------------------------------------------+
   | ``snr conn``          | Whether the device with the provided serial number is connected to the PC with a serial connection. | ``True`` - Connected.                         |
   |                       |                                                                                                     +-----------------------------------------------+
   |                       |                                                                                                     | ``False`` - Not connected.                    |
   +-----------------------+-----------------------------------------------------------------------------------------------------+-----------------------------------------------+
   | ``device``            | Device type, as provided in the :file:`nrf5340_audio_dk_devices.json` file.                         | ``headset`` - Headset.                        |
   |                       |                                                                                                     +-----------------------------------------------+
   |                       |                                                                                                     | ``gateway`` - Gateway.                        |
   +-----------------------+-----------------------------------------------------------------------------------------------------+-----------------------------------------------+
   | ``only reboot``       | Whether the device is to be only reset and not programmed.                                          | ``Not selected`` - No reset.                  |
   |                       | This depends on the ``-r`` parameter in the command, which overrides other parameters.              +-----------------------------------------------+
   |                       |                                                                                                     | ``Selected TBD`` - Only reset requested.      |
   |                       |                                                                                                     +-----------------------------------------------+
   |                       |                                                                                                     | ``Done`` - Reset done.                        |
   |                       |                                                                                                     +-----------------------------------------------+
   |                       |                                                                                                     | ``Failed`` - Reset failed.                    |
   +-----------------------+-----------------------------------------------------------------------------------------------------+-----------------------------------------------+
   |``core app programmed``| Whether the application core is to be programmed.                                                   | ``Not selected`` - Core won't be programmed.  |
   |                       | This depends on the value provided to the ``-c`` parameter (see above).                             +-----------------------------------------------+
   |                       |                                                                                                     | ``Selected TBD`` - Programming requested.     |
   |                       |                                                                                                     +-----------------------------------------------+
   |                       |                                                                                                     | ``Done`` - Programming done.                  |
   |                       |                                                                                                     +-----------------------------------------------+
   |                       |                                                                                                     | ``Failed`` - Programming failed.              |
   +-----------------------+-----------------------------------------------------------------------------------------------------+-----------------------------------------------+
   |``core net programmed``| Whether the network core is to be programmed.                                                       | ``Not selected`` - Core won't be programmed.  |
   |                       | This depends on the value provided to the ``-c`` parameter (see above).                             +-----------------------------------------------+
   |                       |                                                                                                     | ``Selected TBD`` - Programming requested.     |
   |                       |                                                                                                     +-----------------------------------------------+
   |                       |                                                                                                     | ``Done`` - Programming done.                  |
   |                       |                                                                                                     +-----------------------------------------------+
   |                       |                                                                                                     | ``Failed`` - Programming failed.              |
   +-----------------------+-----------------------------------------------------------------------------------------------------+-----------------------------------------------+

.. _nrf53_audio_app_building_standard:

Building and programming using command line
===========================================

You can also build the nRF5340 Audio application using the standard |NCS| :ref:`build steps <gs_programming>` for the command line.

.. note::
   Using this method requires you to build and progam each development kit one at a time before moving to the next configuration, which can be time-consuming.
   :ref:`nrf53_audio_app_building_script` is recommended.

Building the application
------------------------

Complete the following steps to build the application:

1. Choose the combination of build flags:

   a. Choose the device type by using one of the following options:

      * For headset device: ``-DCONFIG_AUDIO_DEV=1``
      * For gateway device: ``-DCONFIG_AUDIO_DEV=2``

   #. Choose the application version by using one of the following options:

      * For the debug version: No build flag needed.
      * For the release version: ``-DCONF_FILE=prj_release.conf``

#. Build the application using the standard :ref:`build steps <gs_programming>`.
   For example, if you want to build the firmware for the application core as a headset using the ``release`` application version, you can run the following command:

   .. code-block:: console

      west build -b nrf5340_audio_dk_nrf5340_cpuapp --pristine -DCONFIG_AUDIO_DEV=1 -DCONF_FILE=prj_release.conf

   Unlike when :ref:`nrf53_audio_app_building_script`, this command creates the build files directly in the :file:`build` directory.
   This means that you first need to program the headset development kits before you build and program gateway development kits.
   Alternatively, you can add the ``-d`` parameter to the ``west`` command to specify a custom build folder. This lets you build firmware for both
   headset and gateway before programming any development kits.

Programming the application
---------------------------

After building the files for the development kit you want to program, complete the following steps to program the application from the command line:

1. Plug the device into the USB port using USB-C.
#. Turn on the development kit using the On/Off switch.
#. Open a command prompt.
#. Run the following command to print the SEGGER serial number of your development kit:

   .. code-block:: console

      nrfjprog -i

   .. note::
      Pay attention to which device is to be programmed with the gateway HEX file and which devices are to be programmed with the headset HEX file.

#. Program the network core on the development kit by running the following command:

   .. code-block:: console

      nrfjprog --program bin/*.hex --chiperase --coprocessor CP_NETWORK -r

   |net_core_hex_note|
#. Program the application core on the development kit with the respective HEX file from the :file:`build` directory by running the following command:

   .. code-block:: console

      nrfjprog --program build/zephyr/zephyr.hex --coprocessor CP_APPLICATION --sectorerase -r

   In this command, :file:`build/zephyr/zephyr.hex` is the HEX binary file for the application core.
   If a custom build folder is specified, the path to this folder must be used instead of :file:`build/`.
#. If any device is not programmed due to :ref:`readback_protection_error`, complete the following steps:

   a. Run the following commands to recover the device:

      .. code-block:: console

         nrfjprog --recover --coprocessor CP_NETWORK
         nrfjprog --recover

   #. Repeat steps 5 and 6 to program both cores again.

#. When using the default CIS configuration, if you want to use two headset devices, you must also populate the UICR with the desired channel for each headset.
   Use the following commands, depending on which headset you want to populate:

   * Left headset:

     .. code-block:: console

        nrfjprog --memwr 0x00FF80F4 --val 0

   * Right headset:

     .. code-block:: console

        nrfjprog --memwr 0x00FF80F4 --val 1

   Select the correct board when prompted with the popup or add the ``--snr`` parameter followed by the SEGGER serial number of the correct board at the end of the ``nrfjprog`` command.

.. _nrf53_audio_app_testing:

Testing
=======

After building and programming the application, you can test it for both the CIS and the BIS modes.
The following testing scenarios assume you are using USB as the audio source on the gateway.
This is the default setting.

.. _nrf53_audio_app_testing_steps_cis:

Testing the default CIS mode
----------------------------

Complete the following steps to test the CIS mode for one gateway and two headset devices:

1. Make sure that the development kits are still plugged into the USB port using USB-C and are turned on.
   After programming, **RGB2** starts blinking green on every device to indicate the ongoing CPU activity on the network core.
   **LED3** starts blinking green on every device to indicate the ongoing CPU activity on the application core.
#. Wait for the **LED1** on the gateway to start blinking blue.
   This happens shortly after programming the development kit and indicates that the gateway device is connected to at least one headset and ready to send data.
#. Search the list of audio devices listed in the sound settings of your operating system for *nRF5340 USB Audio* (gateway) and select it as the output device.
#. Connect headphones to the **HEADPHONE** audio jack on both headset devices.
#. Start audio playback on your PC from any source.
#. Wait for **LED1** to blink blue on both headsets.
   When they do, the audio stream has started on both headsets.

   .. note::
      The audio outputs only to the left channel of the audio jack, even if the given headset is configured as the right headset.
      This is because of the mono hardware codec chip used on the development kits.
      If you want to play stereo sound using one development kit, you must connect an external hardware codec chip that supports stereo.

#. Wait for **LED2** to light up solid green on the headsets to indicate that the audio synchronization is achieved.
#. Press the **VOL+** button on one of the headsets.
   The playback volume increases for both headsets.
#. Press the **VOL-** button on the gateway.
   The playback volume decreases for both headsets.
#. Press the **PLAY/PAUSE** button on one of the headsets.
   The playback stops for the given headset and continues on the other one.
#. Press the **RESET** button on the gateway.
   The gateway resets and the playback on the unpaused headset stops.
   After some time, the gateway establishes the connection with both headsets and resumes the playback on the unpaused headset.
#. Press the **PLAY/PAUSE** button on one of the paused headsets.
   The playback resumes in sync with the other headset.
#. Press the **BTN 4** button on the gateway multiple times.
   For each button press, the audio stream playback is stopped and the gateway sends a test tone to both headsets.
   These tones can be used as audio cues to check the synchronization of the headsets.

When you finish testing, power off the nRF5340 Audio development kits by switching the power switch from On to Off.

.. _nrf53_audio_app_testing_steps_bis:

Testing the BIS mode
--------------------

Testing the BIS mode is identical to `Testing the default CIS mode`_, except for the following differences:

* You must :ref:`select the BIS mode manually <nrf53_audio_app_configuration_select_bis>` before building the application.
* You can play the audio stream with different audio settings on the receivers.
  For example, you can decrease or increase the volume separately for each receiver during playback.

.. _nrf53_audio_app_porting_guide:

Adapting application for end products
*************************************

This section describes the relevant configuration sources and lists the steps required for adapting the nRF5340 Audio application to end products.

Board configuration sources
===========================

The nRF5340 Audio application uses the following files as board configuration sources:

* Devicetree Specification (DTS) files - These reflect the hardware configuration.
  See :ref:`zephyr:dt-guide` for more information about the DTS data structure.
* Kconfig files - These reflect the hardware-related software configuration.
  See :ref:`kconfig_tips_and_tricks` for information about how to configure them.
* Memory layout configuration files - These define the memory layout of the application.

You can see the :file:`nrf/boards/arm/nrf5340_audio_dk_nrf5340` directory as an example of how these files are structured.

For information about differences between DTS and Kconfig, see :ref:`zephyr:dt_vs_kconfig`.
For detailed instructions for adding Zephyr support to a custom board, see Zephyr's :ref:`zephyr:board_porting_guide`.

.. _nrf53_audio_app_porting_guide_app_configuration:

Application configuration sources
=================================

The application configuration source file defines a set of options used by the nRF5340 Audio application.
This is a :file:`.conf` file that modifies the default Kconfig values defined in the Kconfig files.

Only one :file:`.conf` file is included at a time.
The :file:`prj.conf` file is the default configuration file and it implements the debug application version.
For the release application version, you need to include the :file:`prj_release.conf` configuration file.
In the release application version no debug features should be enabled.

The nRF5340 Audio application also use several :file:`Kconfig.defaults` files to change configuration defaults automatically, based on the different application versions and device types.

You need to edit :file:`prj.conf` and :file:`prj_release.conf` if you want to add new functionalities to your application, but editing these files when adding a new board is not required.

.. _nrf53_audio_app_porting_guide_adding_board:

Adding a new board
==================

.. note::
    The first three steps of the configuration procedure are identical to the steps described in Zephyr's :ref:`zephyr:board_porting_guide`.

To use the nRF5340 Audio application with your custom board:

1. Define the board files for your custom board:

   a. Create a new directory in the :file:`nrf/boards/arm/` directory with the name of the new board.
   #. Copy the nRF5340 Audio board files from the :file:`nrf5340_audio_dk_nrf5340` directory located in the :file:`nrf/boards/arm/` folder to the newly created directory.

#. Edit the DTS files to make sure they match the hardware configuration.
   Pay attention to the following elements:

   * Pins that are used.
   * Interrupt priority that might be different.

#. Edit the board's Kconfig files to make sure they match the required system configuration.
   For example, disable the drivers that will not be used by your device.
#. Build the application by selecting the name of the new board (for example, ``new_audio_board_name``) in your build system.
   For example, when building from the command line, add ``-b new_audio_board_name`` to your build command.

Dependencies
************

This application uses the following `nrfx`_ libraries:

* :file:`nrfx_clock.h`
* :file:`nrfx_gpiote.h`
* :file:`nrfx_timer.h`
* :file:`nrfx_dppi.h`
* :file:`nrfx_i2s.h`
* :file:`nrfx_ipc.h`
* :file:`nrfx_nvmc.h`

The application also depends on the following Zephyr libraries:

* :ref:`zephyr:logging_api`
* :ref:`zephyr:kernel_api`
* :ref:`zephyr:api_peripherals`:

   * :ref:`zephyr:usb_api`

* :ref:`zephyr:bluetooth_api`:

  * :file:`include/bluetooth/bluetooth.h`
  * :file:`include/bluetooth/gatt.h`
  * :file:`include/bluetooth/hci.h`
  * :file:`include/bluetooth/uuid.h`

Application configuration options
*********************************

.. options-from-kconfig::
   :show-type:

.. _nrf53_audio_app_dk_legal:

Legal notices and disclaimers
*****************************

Additional Disclaimer for the nRF5340 Audio application
   This application and the LE Audio Controller Subsystem for nRF53 are marked as experimental.
   The LE Audio Controller Subsystem for nRF53 associated with this release comes with QDID 181316.
   This LE Audio link controller is tested and works in configurations used by the present reference code (for example, 2 concurrent CIS, or BIS).
   No other configurations than the ones used in the reference application are tested nor documented in this release.

Important - Battery warnings and mandatory requirements for the nRF5340 Audio DK
   The nRF5340 Audio development kit contains a Rechargeable Li-Po battery with 1500 mAh capacity.
   Please note these warnings and mandatory requirements:

   * The battery in this product shall not be replaced by users themselves.
     Batteries should be removed only by qualified professionals due to safety concerns.

     * Risk of fire or explosion if the battery is replaced by an incorrect type.
     * Disposal of a battery into fire or a hot oven, or mechanically crushing or cutting of a battery can result in an explosion.
     * Leaving a battery in an extremely high temperature surrounding environment can result in an explosion or the leakage of flammable liquid or gas.
     * A battery subjected to extremely low air pressure may result in an explosion or the leakage of flammable liquid or gas.

   * The nRF5340 Audio development kit shall not be operated outside the internal battery's charge & discharge temperature range between +10°C and +60°C or stored or transported outside the internal battery's storage temperature.
   * Power supply adapter must meet PS1 requirements.

   .. figure:: /images/nRF5340_audio_dk_battery_warning.png

Legal notices for the nRF5340 Audio DK
   By using this documentation you agree to our terms and conditions of use.
   Nordic Semiconductor may change these terms and conditions at any time without notice.

   Liability disclaimer
      Nordic Semiconductor ASA reserves the right to make changes without further notice to the product to improve reliability, function, or design.
      Nordic Semiconductor ASA does not assume any liability arising out of the application or use of any product or circuits described herein.

      Nordic Semiconductor ASA does not give any representations or warranties, expressed or implied, as to the accuracy or completeness of such information and shall have no liability for the consequences of use of such information.
      If there are any discrepancies, ambiguities or conflicts in Nordic Semiconductor’s documentation, the Product Specification prevails.

      Nordic Semiconductor ASA reserves the right to make corrections, enhancements, and other changes to this document without notice.

   Life support applications
      Nordic Semiconductor products are not designed for use in life support appliances, devices, or systems where malfunction of these products can reasonably be expected to result in personal injury.

      Nordic Semiconductor ASA customers using or selling these products for use in such applications do so at their own risk and agree to fully indemnify Nordic Semiconductor ASA for any damages resulting from such improper use or sale.

   Radio frequency notice
      The nRF5340 Audio development kit operates in the 2.4 GHz ISM radio frequency band.
      The maximum radio frequency power transmitted in the frequency band in which the development kit operates equals +3dBm (2 mW).

   RoHS and REACH statement
      Complete hazardous substance reports, material composition reports and latest version of Nordic's REACH statement can be found on our website www.nordicsemi.com.

   Trademarks
      All trademarks, service marks, trade names, product names, and logos appearing in this documentation are the property of their respective owners.

   Copyright notice
      © 2022 Nordic Semiconductor ASA.
      All rights are reserved.
      Reproduction in whole or in part is prohibited without the prior written permission of the copyright holder.

.. |lc3_codec_access_note| replace:: The default software codec for the application is LC3, which is not open-source.
   To build the application using this codec requires obtaining access to the LC3 codec repository.
   To obtain access to the repository, `contact the sales department <Contact Us_>`_.

.. |net_core_hex_note| replace:: The network core for both gateway and headsets is programmed with the precompiled Bluetooth Low Energy Controller binary file :file:`ble5-ctr-rpmsg_<XYZ>.hex`, where ``<XYZ>`` corresponds to the controller version, for example :file:`ble5-ctr-rpmsg_3216.hex`.
   This file includes the LE Audio Controller Subsystem for nRF53 and is provided in the :file:`applications/nrf5340_audio/bin` directory.
