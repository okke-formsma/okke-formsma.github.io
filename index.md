# Python3.6 on Macbook Pro M1
## 2021-12-10

Installing python 3.6 on an M1 machine is possible, but it's not trivial. The trick is to use a x86-64 version of python and install all prerequisites using a brew running under x86 too.

The following steps worked for me.

### Set up 'ibrew', or a x86 brew

Install the x86 homebrew version. All x86-64 packages are installed in /usr/local/, while 'normal' brew saves packages in /opt/homebrew/
* `arch -x86_64 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
* `echo "alias ibrew=\"arch -x86_64 /usr/local/bin/brew\"" >> ~/.zshrc`

### Install anaconda x86-64

* `ibrew install anaconda`
* `/usr/local/anaconda3/bin/conda init zsh` (make sure not to use any arm conda version)

### Create python 3.6 x86-64 venv

* `conda create --name venv_py36 python=3.6`
* `ibrew install libpq`
* `export PKG_CONFIG_PATH="/usr/local/opt/libpq/lib/pkgconfig"`
* `export LDFLAGS="-L/usr/local/opt/libpq/lib"`
* `export CPPFLAGS="-I/usr/local/opt/libpq/include"`
* `pip install psycopg2==2.8.6 --force-reinstall --no-cache-dir`
* `ibrew install imagemagick freetype`
* `pip install python-magic`

# Blackmagic on a bluepill
## 2020-06-30

To flash nrfmicro chips, you can use a bluepill with blackmagic installed.

The bluepill has some issues with wrong resistors (I had to change R3 on the bluepill from 100k to 10k ohm). The blackpill with stm32f103 processor does not have this problem, and is recommended. (Black Magic is not supported on blackpill 1.2 (stm32f401) or blackpill 2.0 (stm32f411) at this time.

There's an excellent article on flashing blackmagic firmware onto a bluepill by joric. Here are some the steps I went through to make it work on ubuntu.

Connect the usb-to-uart converter to the bluemicro and set the bluemicro boot0 jumper to 1.

Find the usb port that the converter is attached to

```
> dmesg | grep tty
[157210.247340] cp210x ttyUSB0: cp210x converter now disconnected from ttyUSB0
[157221.140041] usb 1-2: cp210x converter now attached to ttyUSB0
```

Install the dfu. Run stm32loader.py on python2 with the pySerial package installed:
```
python2 stm32loader.py -p /dev/ttyUSB0 -ewv blackmagic_dfu.bin
```
Install the blackmagic firmware:
```
stm32flash -w blackmagic.bin /dev/ttyUSB0 -S 0x08002000
```
Set the jumper back to 0, disconnect the uart adapter, connect the bluepill to usb.
Check if the blackmagic probe is recognized with lsusb.


# Using a trackball as scrollwheel
## 2020-05-27
Tiny scrollwheel on mice have a few problems. They are small and the movement to scroll is prone to RSI. I've been using an Elecom HUGE trackball, and I love it. Except it doesn't have a nice scroll ring as the Kensingtons have. Today I figured out how to turn the entire trackball into a huge scrollwheel by configuring libinput.

To try it out, run a few xinput commands.

* `xinput list` and find the device id of your mouse. In my case it's 14.
* `xinput set-prop 14 "libinput Scroll Method Enabled" 0, 0, 1` to enable 'button scrolling'
* `xinput set-prop 14 "libinput Button Scrolling Button" 12` to use 'Fn3' on the Huge as the scroll trigger. If you want to use the 'middle mouse button', use 2 here.

If you like it, create the file /usr/share/X11/xorg.conf.d/60-huge.conf to make these settings persist between reboots.

```
Section "InputClass"
    Identifier  "ELECOM TrackBall Mouse HUGE TrackBall"
    Option  "ScrollMethod" "button"
    Option  "ScrollButton" "12"
EndSection
```

