# Channel Sounding with USRPs in GNU Radio

This respository contains a minimal working example and some practical guidance for correlation-based channel sounding using [USRPs](https://en.wikipedia.org/wiki/Universal_Software_Radio_Peripheral) in [GNU Radio](https://wiki.gnuradio.org/index.php/Tutorials).

The advantage of correlation-based channel sounding is that transmitter and receiver can be run separately without the need for synchronization, which enables convenient mobile measurement setups.
The disadvantage is that it requires some data post-processing and that you won't be able to measure absolute propagation delays.

The advantage of using USRPs and laptops running GNU Radio is a comparatively affordable and mobile setup.
The disadvantage of using USRPs instead of a calibrated spectrum analyzer or vector network analyzer is reduced dynamic range and that you won't be able to measure absolute power levels.
Also, you need to make sure that there are no other strong transmitters nearby (even far outside your bandwidth of interest) because of the USRP's large analog receive bandwidth.
The latter can be mitigated by getting an analog bandpass filter for your band of interest.


## USRP Bus Series and GNU Radio on Debian/Ubuntu

This section should help you getting started with the portable [USB-type series of USRPs](https://www.ettus.com/product-categories/USRP-Bus-Series/) and GNU Radio.
It assumes that you have a basic understanding of Debian/Ubuntu and software-defined radio in general.
The following has been tested for the USRP B205mini-i and Ubuntu 22.04.

Start by installing UHD (the USRP hardware driver) and GNU Radio:
```bash
sudo apt install uhd_host gnuradio
```

Every time a USB-type USRP is powered on (plugged in), it needs to get an image downloaded to its FPGA before it can be used by GNU Radio.
Download the UHD FPGA images to your system:
```bash
sudo /usr/lib/uhd/utils/uhd_images_downloader.py
```

If you want regular users to be able to access the USRPs, you can install prepared udev rules:
```bash
cd /usr/lib/uhd/utils
sudo cp uhd-usrp.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Further information on setting up your system for USRPs (including devices from the network series) can be found in the [UHD and USRP Manual](https://files.ettus.com/manual/page_transport.html).

Verify that your computer can talk to your USRPs by running `uhd_find_devices`.

Finally, doing signal processing in software consumes a lot of CPU time, and since we want to send and receive signals in realtime, it needs to be done with low latency.
GNU Radio can request realtime scheduling from the Linux kernel, but it needs permission to do so.
The above installation includes an appropriate file in `/etc/security/limits.d/` which will allow users of the group `usrp` to request the highest realtime priority.
Add yourself to this group:
```bash
sudo adduser <yourusername> usrp
```
Log out and back in or reboot your system to apply this change.

You may also want to disable power saving for your CPU while running a measurement, because CPU speed transitions take time and increase latency which may lead to signal corruption due to lost samples.
If you have the package `cpufrequtils` installed, and you have 3 CPU cores, you can do:
```bash
for CPU in {0..3}; do sudo cpufreq-set -g performance -c $CPU; done
```
Verify by running `cpufreq-info` that all CPU cores are in _performance_ mode.
This setting does not persist across reboots.


## GNU Radio Flowgraphs

_Note that your USRP has multiple RF ports.
In the following, the antenna is assumed to be connected to TRX.
Remember to connect an antenna before starting to transmit to prevent damage due to reflected power._

The repository contains three GNU Radio flowgraphs:
- `transmit.grc` for constantly transmitting a sounding reference signal in a loop
- `analyze.grc` for a live preview of the frequency and impulse response
- `capture.grc` for writing chunks of raw data to a file

All flowgraphs contain parameters for center frequency and bandwidth (`samp_rate`).
Upon execution of a flowgraph (click the play button in GNU Radio Companion), there are GUI widgets to change this from the defaults (40 MHz around 3.75 GHz) at runtime.
You can change these defaults permanently by double-clicking the parameter blocks and editing their default value.

The transmit and analyze flowgraphs contain a 255-sample [Zadoff-Chu](https://en.wikipedia.org/wiki/Zadoff%E2%80%93Chu_sequence) sequence serves as sounding reference signal.
It has been padded by 1 sample to match an efficient FFT size of 256 at the cost of slightly reduced autocorrelation sharpness.
You can replace the contents of the complex-valued vector with any sequence of your choice.

The analyze flowgraph correlates the input with the reference signal according to the discrete correlation theorem.
Its output (corresponding to an estimate of the channel's impulse response without frequency correction) is plotted in time and frequency domain.

The capture flowgraph saves raw [32-bit floating point IQ data](https://wiki.gnuradio.org/index.php/File_Sink#Handling_File_Sink_data) to a timestamped file for later post-processing.
Saving all samples may be excessive if the channel changes slowly compared to the considered bandwidth.
Hence, by default, only chunks of 2048 samples (spanning up to 8 complete repetitions of the reference signal) are saved every 10 milliseconds.
This should be considered in the post-processing.
Besides, the flowgraph features live plots of the raw input in time and frequency domain.
Make sure that the input level never reaches 0 dB in the time domain plot, otherwise there will be clipping.
It's a good idea to keep at least 10 dB headroom even under good propagation conditions (the level may still increase due to constructive reflections).

Especially for bandwidths of 40 MHz and above, you may need a powerful computer to avoid missing samples and thereby destroying your measurement.
Watch out for underrun (U) and overflow (O) messages in the bottom left log panel in GNU Radio Companion.
Getting overflows in the analyze flowgraph doesn't matter, but any underrun in the transmit flowgraph and any overflow in the capture flowgraph will lead to corrupted data!
Check that GNU Radio was actually able to obtain realtime scheduling (it will tell you at the beginning of the log) and that your CPU is not in powersave mode (see previous section).
