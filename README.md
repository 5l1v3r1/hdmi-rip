# hdmi-rip
Video/Audio ripper for HDMI IP network extender.

This code will take the output of a LENKENG HDMI over IP extender (Version 2.0) and save it to raw audio (signed 32 bit big-endian PCM) and MJPEG video (YUV 4:2:2).

The hardware can be bought cheaply on ebay by searching for 'HDMI Extender 120m', and the device should look like this:

![LAN extender](extender.jpg)

Make sure it says "LAN/TX" on the CAT5 socket, not just TX as there are some models that do not use IP, but only a straight cable connection. Also, make sure it is the V2.0 model, not V3. The receiver is not required so if you can find a TX only unit cheaper, all the better.

Despite the cheapness of the device, the video output is very high quality. This is because it is effectively dumping a frame buffer direct from HDMI and is using the YUV 4:2:2 Chroma scheme so there is very little noticeable degredation. The audio, on the other hand, is always transcoded to raw PCM Stereo, so you will lose any surround sound etc.

##Installation:

```

  sudo python ./setup.py install

```

##Usage:

```

  hdmi-rip.py [options] <file prefix> [minutes]

  Options:
    -h, --help            show this help message and exit
    -a AUDIO_RATE, --audio_rate=AUDIO_RATE
                          audio data rate in Hz (48000)
    -c AUDIO_CHANNELS, --audio_channels=AUDIO_CHANNELS
                          audio channels (2)
    -f FRAME_RATE, --frame_rate=FRAME_RATE
                          video frame rate (29.97)
    -H HEIGHT, --height=HEIGHT
                          monitor window height (540)
    -k, --keyboard        start/stop recording under keyboard control (False)
    -l LOCAL_IP, --local_ip=LOCAL_IP
                          use local IP address as source (0.0.0.0)
    -n, --no_monitor      do not monitor video in pop-up window (False)
    -p SENDER_PORT, --sender_port=SENDER_PORT
                          set sender's UDP PORT (48689)
    -q, --quiet           do not print status messages to stdout (False)
    -s SENDER_IP, --sender_ip=SENDER_IP
                          set sender's IP address (192.168.168.55)
    -S, --strict          strict mode - abort recording if frames dropped
                          (False)
    -w, --wave            save audio in .wav format (False)
    -W WIDTH, --width=WIDTH
                          monitor window width (960)

```

You will need to run as root to be able create the appropriate network sockets, and have a local IP address on the same network as the sender. You must also have a default route set in order to be able to join the multicast group.

##Example:

```

  sudo ifconfig eth0:1 inet 192.168.168.1

  if direct connection:

     sudo route add default gw 192.168.168.1 

  sudo hdmi-rip.py /tmp/dummy 20

```

  Will produce /tmp/dummy-video.dat and /tmp/dummy-audio.dat and automatically stop after 20 minutes.

##Transcoding:

  I do this in two stages. The first synchronises the audio but leaves the video alone which speeds up processing. This output is then used for editing/trimming. Adjust the audio offset with '-itsoffset'

  For NTSC content no audio offset:

```

    ffmpeg -f mjpeg -r 29.97 -i /tmp/dummy-video.dat -itsoffset 0.0 -f s32be -ac 2 -ar 48000 -i /tmp/dummy-audio.dat -f matroska -vcodec copy -c:a libmp3lame -qscale:a 2  /tmp/dummy.mkv

```

  For PAL content, audio delayed by 0.2 seconds:

```

    ffmpeg -f mjpeg -r 25 -i /tmp/dummy-video.dat -itsoffset 0.2 -f s32be -ac 2 -ar 48000 -i /tmp/dummy-audio.dat -f matroska -vcodec copy -c:a libmp3lame -qscale:a 2  /tmp/dummy.mkv

```

  I use kdenlive to trim: add clip to project then select entry and exit points in clip monitor and right-click to 'extract zone' for a lossless edit.
  
  Once trimmed, transcode the video:

```

    ffmpeg -i /tmp/dummy.mkv -c:v mpeg4 -vtag xvid -qscale:v 2 -c:a copy /tmp/final.mkv

```

  The final output of a 720p capture of Big Buck Bunny, (c) copyright 2008, Blender Foundation / www.bigbuckbunny.org, can be viewed here: 

  <a href="http://rfidiot.org/hdmi-mjpeg/bb-rip-720-trimmed-final.mkv" target="_blank"><img src="http://rfidiot.org/hdmi-mjpeg/bbb-1024.png" alt="Big Buck Bunny" width="1024" height="768" border="10"></a>

##Tips & tricks:

  Reduce packet loss by using a direct connection instead of a switch (don't forget to set a dummy default route or you won't be able to join the multicast group).

  Seemingly unrelated actions like heavy load from other processes or even screensavers can cause packet loss.

  Adding a dual output HDMI splitter will allow you to monitor the source while setting up for recording. It also prevents auto-switching of HDMI settings sometimes caused by switching cables.

  You may get a better soundtrack by switching your source to Stereo/PCM instead of allowing the transmitter to downscale (if you use a home-theater amp, you'll need to configure the amp instead and only send Stereo/PCM audio to the TV so you can tap that output with the splitter and capture from any source).

##Further development you could help with:

  - Add support for other senders.

##Credits:

  This project was inspired by benjojo's blog: https://blog.benjojo.co.uk/post/cheap-hdmi-capture-for-linux which was in turn inspired by danman's blog: https://blog.danman.eu/reverse-engineering-lenkeng-hdmi-over-ip-extender/

  Original script by Silver Moon (m00n.silv3r@gmail.com) and danman.

  Framerates by Jared Earl. :)

##Notes:

  Video frames are buffered and discarded if any part of the frame is lost or received out of sequence. This is safe to do as the data is in MJPEG format so every frame is a complete image, and loss of one frame every now and then will not be noticeable. The same cannot be said for audio as although it is discarded in the same way (and also during video packet loss to maintain sync) the effect is definitely more noticeable.

  The audio track is currently not tagged with its data rate, so you may need to experiment to decide what it is. The only format I've seen so far is 48KHz which gives the above video framerates for transcoding. If your soundtrack in the monitor window goes out of sync then you've probably used the wrong one (despite being in the UK, a lot of streaming services are actually transmitted with NTSC framerates). Any changes you make to the monitoring settings will have to be re-applied during transcoding as the output files are still the original raw stream.

  HDCP is stripped by the hardware. Why this is not a surprise: http://adamsblog.aperturelabs.com/2013/02/hdcp-is-dead-long-live-hdcp-peek-into.html

  YMMV: the output quality appears to be quite dependant on the input format. I've seen 1080P rips that are almost indistinguishable from the original, and yet the 720P Big Buck Bunny rip above has noticeable artifacts which are introduced by the sender, not the 2nd stage transcoding.

##Copyrighted material

  Where would we be without the usual waffle about copyright?

  In the UK we have a thing called "Time-shifting", the details of which can be viewed here: https://www.gov.uk/guidance/exceptions-to-copyright

  In my view, this gives me the right to record anything and everything as long as I don't re-distribute it and only use it within my own houeshold/family non-commercially. However, I'm not a lawyer so take your own advice and bear in mind that kittens, unicorns and/or other mythical creatures may be harmed if you do not respect Copyright in both letter and spirit, and you may make the baby Jesus cry.

  For clarity: this code is published for the sole purpose of legally recording content that has had NotVeryGood(tm) copy-protection applied to it. Please do not abuse that facility, and enjoy your recordings in the comfort and safety of your own home(s).
