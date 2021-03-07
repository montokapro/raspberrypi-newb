## Node exporter

I installed according to the steps here, slightly modified: https://jorge.maroto.me/monitoring-raspberry-prometheus-node_explorer/

```
export NODE_EXPORTER_VERSION="1.1.2"

cd ~/Downloads

# Download the package
wget https://github.com/prometheus/node_exporter/releases/download/v$NODE_EXPORTER_VERSION/node_exporter-$NODE_EXPORTER_VERSION.linux-armv7.tar.gz

# Extract the package
sudo tar -xvf node_exporter-$NODE_EXPORTER_VERSION.linux-armv7.tar.gz -C /usr/local/bin/ --strip-components=1

# Run the node_exporter
node_exporter
```

Then I modified crontab. Consider switching to shepherd or systemd in the future.

```
crontab -e

# Add the following line to your crontab and save the file
@reboot /usr/local/bin/node_exporter
```

I still need a way to view these metrics with prometheus and grafana.

## Sound

We can stream audio from the pi
```
echo ffmpeg -re -i BabyElephantWalk60.wav -acodec pcm_s16be -ar 44100 -ac 1 -f rtp rtp://192.168.0.237:5555
```

The following prove that we are selecting the correct device:
```
arecord -l
arecord -L
arecord -f cd -D plughw:1,0 > /dev/null
```

The following works but is too quiet:
```
ffmpeg -re -ac 1 -f alsa -i hw:1,0 -acodec libmp3lame -ab 32k -ac 1 -f rtp rtp://192.168.0.237:5555
```

With alsamixer, I can increase the volume of the microphone. However, this adds a lot of static.

I can use ffmpeg to filter noise:
```
ffmpeg -re -ac 1 -f alsa -i hw:1,0 -acodec libmp3lame -ar 44100 -ac 1 -af "afftdn=nf=-25" -f rtp rtp://192.168.0.237:5555
```

I can adjust volume as well, but the noise is amplified:
```
ffmpeg -re -ac 1 -f alsa -i hw:1,0 -acodec libmp3lame -ar 44100 -ac 1 -af "volume=0.01" -af "afftdn=nf=-25" -f rtp rtp://192.168.0.237:5555
```

## Camera

I verified the camera is installed by importing the library in python:
```
python3 -c "import picamera"
```

I can see all available devices with the following:
```
v4l2-ctl --list-devices

bcm2835-codec-decode (platform:bcm2835-codec):
	/dev/video10
	/dev/video11
	/dev/video12

mmal service 16.1 (platform:bcm2835-v4l2):
	/dev/video0
```

I attempted to stream using the camera with ffmpeg
```
ffmpeg -f v4l2 -i /dev/video0 -framerate 15 -pix_fmt yuvj420p -level:v 4.1 -vcodec libx264 -r 10 -b:v 128k -s 640x360 -f mpegts rtp://192.168.0.237:5555?ttl=10
```

This didn't crash on the pi side, but was unable to decode on client VLC. I looked for examples, and found https://gist.github.com/moritzmhmk/48e5ed9c4baa5557422f16983900ca95

However, I was able to use UDP:
```
ffmpeg -f video4linux2 -i /dev/video0 -vf scale=1280:720 -vcodec libx264 -pix_fmt yuv420p -tune zerolatency -preset ultrafast -f mpegts udp://192.168.0.237:5555
```

## Putting it all together

The following command works to add the audio alongside the video
```
ffmpeg \
-re -ac 1 \
-f alsa -i hw:1,0 \
-f video4linux2 -i /dev/video0 \
-acodec libmp3lame -ar 44100 -af "afftdn=nf=-25" \
-vf scale=1280:720 -vcodec libx264 -pix_fmt yuv420p -tune zerolatency -preset ultrafast \
-f mpegts udp://192.168.0.237:5555
```
