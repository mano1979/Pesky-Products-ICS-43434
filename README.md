<html><head></head><body><div class="content">
<h1>Setting up the Pesky Products ICS-43434 MEMS microphone breakout on the Raspberry Pi</h1><br><br>
<h2>Hardware Setup</h2>
<p>The following documentation used the ICS43434 MEMs microphone with a breakout board on an RPi 2.  Microphone documentation can be found <a href="https://www.embeddedmasters.com/datasheets/embedded/EMMIC-ICS43432-DS.pdf">here</a>.  Header pins were soldered to the breakout board.  The connection uses Pulse Code Modulation which requires four GPIO pins from the RPi.  The PCM setup can be found <a href="https://pinout.xyz/pinout/pcm">here</a>.  The connection is as follows:
<a href="https://raw.githubusercontent.com/nejohnson2/rpi-i2s/master/rpi-pins.png" target="_blank"><img src="https://raw.githubusercontent.com/nejohnson2/rpi-i2s/master/rpi-pins.png" align="right" width="334" height="446" style="max-width:100%;"></a></p>
<pre><code>Mic - RPi
---------
VCC - 3.3v
Gnd - Gnd
L/R - Gnd (this is used for channel selection. Connect to 3.3 or GND)
SCK - BCM 18 (pin 12)
WS  - BCM 19 (pin 35)
SD  - BCM 20 (pin 38)
</code></pre>
<p><a href="https://www.tindie.com/products/onehorse/ics43434-i2s-digital-microphone/" target="_blank"><img src="https://cdn.tindiemedia.com/images/resize/zAeH834QzxKpn1sKrsz0R3rg4bw=/900x600/smart/44691/products/2017-07-15T04%3A32%3A01.633Z-ICS43434.top.jpg" alt="Pesky Products ICS-43434 Breakoutboard" width="150" style="max-width:10%;"><br>Buy your Pesky Products ICS-43434 breakout board here</a></p>
<br><br>
<h2>Software Setup</h2>
Firstly, get an updated kernel &amp; matching kernel header files:
<br><br><pre><code>sudo apt-get update
sudo apt-get dist-upgrade
sudo apt-get install raspberrypi-kernel-headers
sudo reboot
</code></pre></div>
Next, while the upstream ics43432 codec is not currently supported by current Pi kernel builds, we must it build manually.
<br>
Get the source &amp; create a simple Makefile:<pre><code>mkdir ics43432
cd ics43432
wget https://raw.githubusercontent.com/raspberrypi/linux/rpi-4.4.y/sound/soc/codecs/ics43432.c
nano Makefile</code></pre></div>
Makefile contents:<pre><code>

obj-m := ics43432.o

all:
&#9;make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
&#9;make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

install:
&#9;sudo cp ics43432.ko /lib/modules/$(shell uname -r)
&#9;sudo depmod -a
</code></pre></div>Those indentations are a single tab character - use spaces &amp; it won't work.<br>
<br>
Build:<pre><code>make all install</code></pre></div>

Next, we create a new device tree overlay. Create a file i2s-soundcard-overlay.dts with this content:<pre><code>
/dts-v1/;
/plugin/;

/ {
&#9;compatible = "brcm,bcm2708";

&#9;fragment@0 {
&#9;&#9;target = &lt;&amp;i2s&gt;;
&#9;&#9;__overlay__ {
&#9;&#9;&#9;status = "okay";
&#9;&#9;};
&#9;};

&#9;fragment@1 {
&#9;&#9;target-path = "/";
&#9;&#9;__overlay__ {
&#9;&#9;&#9;card_codec: card-codec {
&#9;&#9;&#9;&#9;#sound-dai-cells = &lt;0&gt;;
&#9;&#9;&#9;&#9;compatible = "invensense,ics43432";
&#9;&#9;&#9;&#9;status = "okay";
&#9;&#9;&#9;};
&#9;&#9;};
&#9;};

&#9;fragment@2 {
&#9;&#9;target = &lt;&amp;sound&gt;;
&#9;&#9;master_overlay: __dormant__ {
&#9;&#9;&#9;compatible = "simple-audio-card";
&#9;&#9;&#9;simple-audio-card,format = "i2s";
&#9;&#9;&#9;simple-audio-card,name = "soundcard";
&#9;&#9;&#9;simple-audio-card,bitclock-master = &lt;&amp;dailink0_master&gt;;
&#9;&#9;&#9;simple-audio-card,frame-master = &lt;&amp;dailink0_master&gt;;
&#9;&#9;&#9;status = "okay";
&#9;&#9;&#9;simple-audio-card,cpu {
&#9;&#9;&#9;&#9;sound-dai = &lt;&amp;i2s&gt;;
&#9;&#9;&#9;};
&#9;&#9;&#9;dailink0_master: simple-audio-card,codec {
&#9;&#9;&#9;&#9;sound-dai = &lt;&amp;card_codec&gt;;
&#9;&#9;&#9;};
&#9;&#9;};
&#9;};

&#9;fragment@3 {
&#9;&#9;target = &lt;&amp;sound&gt;;
&#9;&#9;slave_overlay: __overlay__ {
&#9;&#9;&#9;&#9;compatible = "simple-audio-card";
&#9;&#9;&#9;&#9;simple-audio-card,format = "i2s";
&#9;&#9;&#9;&#9;simple-audio-card,name = "soundcard";
&#9;&#9;&#9;&#9;status = "okay";
&#9;&#9;&#9;&#9;simple-audio-card,cpu {
&#9;&#9;&#9;&#9;&#9;sound-dai = &lt;&amp;i2s&gt;;
&#9;&#9;&#9;&#9;};
&#9;&#9;&#9;&#9;dailink0_slave: simple-audio-card,codec {
&#9;&#9;&#9;&#9;&#9;sound-dai = &lt;&amp;card_codec&gt;;
&#9;&#9;&#9;&#9;};
&#9;&#9;};
&#9;};

&#9;__overrides__ {
&#9;&#9;alsaname = &lt;&amp;master_overlay&gt;,"simple-audio-card,name",
&#9;&#9;&#9;&#9;&lt;&amp;slave_overlay&gt;,"simple-audio-card,name";
&#9;&#9;compatible = &lt;&amp;card_codec&gt;,"compatible";
&#9;&#9;master = &lt;0&gt;,"=2!3";
&#9;};
};
</code></pre></div>

Compile &amp; install the overlay:<pre><code>dtc -@ -I dts -O dtb -o i2s-soundcard.dtbo i2s-soundcard-overlay.dts
sudo cp i2s-soundcard.dtbo /boot/overlays</code></pre></div>

Finally, invoke usage by adding this to /boot/config.txt:<pre><code>dtoverlay=i2s-soundcard,alsaname=mems-mic</code></pre></div>

Reboot and you'll have a microphone recording device<div class="codebox"><p><pre><code>pi@raspberrypi:~arecord -l
**** List of CAPTURE Hardware Devices ****
card 1: memsmic [mems-mic], device 0: bcm2835-i2s-ics43432-hifi ics43432-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0</code></pre></div>
  <br>
Make a 10 second long recording with:<br><pre><code>arecord -Dhw:1 -c2 -r48000 -fS32_LE -twav -d10 -Vstereo test.wav</code></pre></div>
To play the recorded wave-file:
<br><pre><code>aplay test.wav</code></pre>

The volume level is low, and with just one module attached is in one channnel only, silence in the other. Both those issues can be addressed by modifying .asoundrc, see below.<br>
<br>
The overlay is flexible enough to use other codecs by way of the compatible= override. If your preferred codec, a simple ADC or DAC, has a .compatible string defined, use that "manufacturer,chipset" string in the override. There's a switch "master" too if your codec is capable of working as clock master instead of the default slave (untested).</div>


To create the soft mixer for volume control I used <a href="https://www.raspberrypi.org/forums/viewtopic.php?f=38&amp;t=85845" class="postlink">plugh's work</a>, extending it so that we produce true mono/single channel output. Note that I've configured my mic as left - if yours is right you'll have to edit the mic_mono multi section. If you have a stereo pair, you may omit the last section where mic_mono is defined. Edit the alsa config:<pre><code>cd ~
nano .asoundrc</code></pre></div>And replace the content of the file with:<pre><code>pcm.mic_hw{
        type hw
        card memsmic
        channels 2
        format S32_LE
}

pcm.mic_sv{
        type softvol
        slave.pcm mic_hw
        control {
                name "Boost Capture Volume"
                card memsmic
        }
        min_dB -3.0
        max_dB 50.0
}</code></pre></div>

To activate the mixer, we must first make a recording using the newly configured mic_sv pcm:<pre><code>arecord -Dmic_sv -c2 -r48000 -fS32_LE -twav -d10 -Vstereo test.wav</code></pre></div>
Now we can tinker with the boost control. In a terminal, run alsamixer, press F6, select device "mems-mic", then F4 for capture controls. A useful boost is 20-30dB.<br>
A mixer is also in the desktop's Audio Device Settings.<br><br><br>

Now you can record again, but with the new volume settings:
<pre><code>arecord -Dmic_sv -c2 -r48000 -fS32_LE -twav -d10 -Vstereo test.wav</code></pre>

To play the recorded wave-file:
<br><pre><code>aplay test.wav</code></pre>

Most of this info came from: https://www.raspberrypi.org/forums/viewtopic.php?t=173640
</body></html>
