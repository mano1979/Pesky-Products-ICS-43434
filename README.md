<html><head></head><body>Source: https://www.raspberrypi.org/forums/viewtopic.php?t=173640
<div class="content">
<h1>Setting up the Pesky Products ICS-43434 MEMS microphone breakout on the Raspberry Pi</h1><br><br>
<h2>Hardware Setup</h2>
<p>The following documentation used the ICS43432 MEMs microphone with a breakout board on an RPi 2.  Mirophone documentation can be found <a href="https://www.embeddedmasters.com/datasheets/embedded/EMMIC-ICS43432-DS.pdf">here</a>.  Header pins were soldered to the breakout board.  Unfortunately the breakout board was poorly designed and in order to properly install the header pins, the pin labels were covered.  Regardless, the connection uses Pulse Code Modulation which requires four GPIO pins from the RPi.  The PCM setup can be found <a href="https://pinout.xyz/pinout/pcm">here</a>.  The connection is as follows:
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
<p><a href="/nejohnson2/rpi-i2s/blob/master/incs43432_breakout.png" target="_blank"><img src="/nejohnson2/rpi-i2s/raw/master/incs43432_breakout.png" alt="INCS43432 Breakoutboard" style="max-width:100%;"></a></p>
<br>
For software, you can either follow the steps there, or do it the modern way here using a device tree overlay.<br>
<br>
Firstly, get an updated kernel &amp; matching kernel header files:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>sudo apt-get update
sudo apt-get dist-upgrade
sudo apt-get install raspberrypi-kernel-headers
sudo reboot
</code></pre></div>
Next, while the upstream ics43432 codec is not currently supported by current Pi kernel builds, we must it build manually.<br>
<br>
Get the source &amp; create a simple Makefile:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>mkdir ics43432
cd ics43432
wget https://raw.githubusercontent.com/raspberrypi/linux/rpi-4.4.y/sound/soc/codecs/ics43432.c
nano Makefile</code></pre></div>
Makefile contents:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>obj-m := ics43432.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

install:
	sudo cp ics43432.ko /lib/modules/$(shell uname -r)
	sudo depmod -a
</code></pre></div>Those indentations are a single tab character - use spaces &amp; it won't work.<br>
<br>
Build:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>make all install</code></pre></div>

Next, we create a new device tree overlay. Create a file i2s-soundcard-overlay.dts with this content:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>/dts-v1/;
/plugin/;

/ {
    compatible = "brcm,bcm2708";

    fragment@0 {
        target = &lt;&amp;i2s&gt;;
        __overlay__ {
            status = "okay";
        };
    };

    fragment@1 {
        target-path = "/";
        __overlay__ {
            card_codec: card-codec {
                #sound-dai-cells = &lt;0&gt;;
                compatible = "invensense,ics43432";
                status = "okay";
            };
        };
    };

    fragment@2 {
        target = &lt;&amp;sound&gt;;
        master_overlay: __dormant__ {
            compatible = "simple-audio-card";
            simple-audio-card,format = "i2s";
            simple-audio-card,name = "soundcard";
            simple-audio-card,bitclock-master = &lt;&amp;dailink0_master&gt;;
            simple-audio-card,frame-master = &lt;&amp;dailink0_master&gt;;
            status = "okay";
            simple-audio-card,cpu {
                sound-dai = &lt;&amp;i2s&gt;;
            };
            dailink0_master: simple-audio-card,codec {
                sound-dai = &lt;&amp;card_codec&gt;;
            };
        };
    };

    fragment@3 {
        target = &lt;&amp;sound&gt;;
        slave_overlay: __overlay__ {
                compatible = "simple-audio-card";
                simple-audio-card,format = "i2s";
                simple-audio-card,name = "soundcard";
                status = "okay";
                simple-audio-card,cpu {
                    sound-dai = &lt;&amp;i2s&gt;;
                };
                dailink0_slave: simple-audio-card,codec {
                    sound-dai = &lt;&amp;card_codec&gt;;
                };
        };
    };

    __overrides__ {
        alsaname = &lt;&amp;master_overlay&gt;,"simple-audio-card,name",
                    &lt;&amp;slave_overlay&gt;,"simple-audio-card,name";
        compatible = &lt;&amp;card_codec&gt;,"compatible";
        master = &lt;0&gt;,"=2!3";
    };
};
</code></pre></div>

Compile &amp; install the overlay:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>dtc -@ -I dts -O dtb -o i2s-soundcard.dtbo i2s-soundcard-overlay.dts
sudo cp i2s-soundcard.dtbo /boot/overlays</code></pre></div>

Finally, invoke usage by adding this to /boot/config.txt:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>dtoverlay=i2s-soundcard,alsaname=mems-mic</code></pre></div>

Reboot and you'll have a microphone recording device<div class="codebox"><p><pre><code>pi@raspberrypi:~arecord -l
**** List of CAPTURE Hardware Devices ****
card 1: memsmic [mems-mic], device 0: bcm2835-i2s-ics43432-hifi ics43432-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0</code></pre></div>
  <br>
Make a 10 second long recording with:<br><pre><code>arecord -Dhw:1 -c2 -r48000 -fS32_LE -twav -d10 -Vstereo test.wav</code></pre></div>

The volume level is low, and with just one module attached is in one channnel only, silence in the other. Both those issues can be addressed by modifying .asoundrc, see below.<br>
<br>
The overlay is flexible enough to use other codecs by way of the compatible= override. If your preferred codec, a simple ADC or DAC, has a .compatible string defined, use that "manufacturer,chipset" string in the override. There's a switch "master" too if your codec is capable of working as clock master instead of the default slave (untested).</div>


To create the soft mixer for volume control I used <a href="https://www.raspberrypi.org/forums/viewtopic.php?f=38&amp;t=85845" class="postlink">plugh's work</a>, extending it so that we produce true mono/single channel output. Note that I've configured my mic as left - if yours is right you'll have to edit the mic_mono multi section. If you have a stereo pair, you may omit the last section where mic_mono is defined. Edit the alsa config:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>cd ~
nano .asoundrc</code></pre></div>&amp; add:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>pcm.mic_hw{
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

To activate the mixer, we must first make a recording using the newly configured mic_sv pcm:<div class="codebox"><p>Code: <a href="#" onclick="selectCode(this); return false;">Select all</a></p><pre><code>arecord -Dmic_sv -c2 -r48000 -fS32_LE -twav -d10 -Vstereo test.wav</code></pre></div>
Now we can tinker with the boost control. In a terminal, run alsamixer, press F6, select device "mems-mic", then F4 for capture controls. A useful boost is 20-30dB.<br>
A mixer is also in the desktop's Audio Device Settings.<br>
</body></html>
