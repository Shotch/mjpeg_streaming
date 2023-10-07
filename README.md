# Introduction

This repository provides some steps and necessary resources to set up a low-latency, HTTP-accessible video stream using FFmpeg and FFserver. In my instance for teleoperated robotics.
The primary goal is to natively stream from a camera without any re-encoding, making it efficient and fast. The steps have been tested on Raspbian, but should be applicable to other Linux distributions.

### **Disclaimer**
Binding the server to `0.0.0.0` will make it accessible from any IP. Please ensure you understand the security implications, especially if used outside of an intranet setting.

---

## Installation & Setup

### 1. Install Essential Utilities

To interact with video devices and get information about them, we need `v4l-utils`:

`sudo apt-get install v4l-utils`

<br>

Check available devices with:

`v4l2-ctl --list-devices`

<br>

To list the supported formats of your video device:

`v4l2-ctl --device=/dev/video0 --list-formats-ext`
* in this case /dev/video0, but will vary

<br>

### 2. Install FFmpeg Dependencies

Before building FFmpeg, we need to install some dependencies:

For audio and video codecs:

`libmp3lame-dev`, `libopus-dev`, `libx264-dev`

Essential utilities for building and compiling FFmpeg from source:

`autoconf`, `automake`, `build-essential`, `cmake`, `git-core`, `pkg-config`, `texinfo`, `wget`, `zlib1g-dev` 

To do so, un the following:

```bash
sudo apt-get update -qq && sudo apt-get -y install libmp3lame-dev libopus-dev libx264-dev \
  autoconf \
  automake \
  build-essential \
  cmake \
  git-core \
  pkg-config \
  texinfo \
  wget \
  zlib1g-dev
```

<br>

### 3. Build FFmpeg with FFserver

We'll be using FFmpeg version 3.4.6, one of the last versions that included FFserver:

```
mkdir -p ~/ffmpeg_sources && \
cd ~/ffmpeg_sources && \
wget -O ffmpeg-3.4.6.tar.bz2 https://ffmpeg.org/releases/ffmpeg-3.4.6.tar.bz2 --no-check-certificate && \
tar xjvf ffmpeg-3.4.6.tar.bz2 && \
mv ffmpeg-3.4.6 ffmpeg && \
cd ffmpeg && \
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure --enable-ffserver && \
PATH="$HOME/bin:$PATH" make -j4 && \
make -j$(nproc) install && \
hash -r
```
* The `--no-check-certificate` is a little spicy, but that's ffmpeg.org for ya.

<br>

### 4. Configure FFserver

1. Edit the configuration / Make the FFserver config file:

```bash
sudo nano /etc/ffserver.conf
```

2. Add your desired config setup, here's an example:

```xml
HTTPPort 8090
HTTPBindAddress 0.0.0.0
MaxHTTPConnections 100
MaxClients 50
MaxBandwidth 10000000

<Feed cam0.ffm>
File /tmp/cam0.ffm
FileMaxSize 75M
Launch ./ffmpeg -input_format mjpeg -i /dev/video0 -c:v copy -override_ffserver
</Feed>

<Stream cam0>
Feed cam0.ffm
Format mpjpeg
VideoSize 1280x720
VideoFrameRate 30
VideoBitRate 300000
VideoQMin 1
VideoQMax 10
NoAudio
</Stream>

<Stream status>
Format status
</Stream>
```

Also the old docs for reference: https://trac.ffmpeg.org/wiki/ffserver

---

## Starting the Stream

1. Start FFserver:

```bash
sudo ffserver
```
* Note: Optionally add & at the end to continue with ffmpeg in the same terminal, but for testing I'd recommend doing this with two CLI instances for logging purposes.

2. Start the video stream:

```
ffmpeg -framerate 30 -video_size 1280x720 -input_format mjpeg -i /dev/video0 -c:v copy http://localhost:8090/cam0.ffm -override_ffserver
```
* Note: Natively ffserver does not know what to do with the copy codec, and will force the result to do transcoding, the `-override_ffserver` flag fixes that real quick.

* Note: You'll want to compare the output here to the results returned by `v4l2-ctl --device=/dev/video0 --list-formats-ext` above, and then also your `ffserver.conf` Observing the ffmpeg output on startup of this command should give you a few signs of what's going on.

The streams now should be accessible from anywhere on the network by visiting:
View the ffserver status at : `http://<streaming device IP address>:8090/status`
View your stream at: `http://<streaming device IP address>:8090/cam0`

---

## Automating the Stream on Boot

Edit the crontab:

```bash
sudo crontab -e
```

Add the following lines:
```
@reboot ffserver
@reboot ffmpeg -framerate 30 -video_size 1280x720 -input_format mjpeg -i /dev/video0 -c:v copy http://localhost:8090/cam0.ffm -override_ffserver
```
* Note Add multiple feeds, cameras, etc in the config, add the corresponding commands here.
* Likely also worth putting in a python script or otherwise with a watchdog to automatically restart the processes 
