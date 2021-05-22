# Senior Project Documentation

This is the documentation for my ASD senior project. I worked on a prototype that would detect and classify traffic signs, and would detect speeding with the vehicle diagnostics system (OBD-II).

![Monkey](monkey.gif)

## Parts Used

## The Nvidia Jetson Nano

The Nvidia Jetson Nano is the board I decided to use for my senior project. It has a much more powerful GPU than the Raspberry Pi, and therefore works well in GPU-intensive applications, such as machine learning. The Nvidia Jetson Nano isn't a technically a single board computer, as it has two components: the carrier board and the compute/system on a chip (SoC) module. It does share the same 40 pin GPIO layout as the Raspberry Pi, but the current output is lower, and Hardware Pulse Width Modulation (PWM) is disabled by default. A networking card is needed to wireless connectivity and Bluetooth. It can run off of USB power, but it will only draw up to 2 Amps through USB. The SoC module itself requires 2 Amps to run without throttling performance, so ideally the board needs about 3 Amps, as the carrier board requires additional power. The Jetson Nano can run Nvidia's JetPack software, which is based on Ubuntu 18.04. The Jetson Nano Developer Kit boots from a MicroSD, similar to a Raspberry Pi and other single board computers. 

![Jetson Nano Image](./jetson_nano.jpg)

## Physical Assembly

## OS Installation

You can get the image needed from [here](https://developer.nvidia.com/embedded/jetpack#install). Go to the SD Card Image Method subsection, and select the Jetson Nano. It should look like this:

![Jetson Software Download](./jetson_download.png)

As you can see, the images are different depending on the RAM. I did this project with the Jetson Nano 4GB, since that was the only model that existed when I started the project. It theoretically is possible to do this project with the Jetson Nano 2GB, but it will have to swap more aggressively, especially if you do the object detection training on the board. Anyways, just get the image you need, as getting the wrong image will likely render the board unbootable.

[Nvidia has a good guide on flashing the image with to the SD with Etcher.](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit#write) I didn't use Etcher and used dd instead. dd comes with MacOS and most Linux distributions. Using dd doesn't require a GUI, but be careful if you do this approach! Etcher has protections set in place to avoid overwriting your main drive, while dd does not. If you're doing the dd approach, you can find the drive with fdisk:

```
$ sudo fdisk -l
Disk /dev/nvme1n1: 238.47 GiB, 256060514304 bytes, 500118192 sectors
Disk model: SKHynix_HFS256GD9TNI-L2B0B              
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 82FD43FC-4343-417E-A104-CC28E18B9347

Device             Start       End   Sectors   Size Type
/dev/nvme1n1p1      2048    534527    532480   260M EFI System
/dev/nvme1n1p2    534528    567295     32768    16M Microsoft reserved
/dev/nvme1n1p3    567296 498069503 497502208 237.2G Microsoft basic data
/dev/nvme1n1p4 498069504 500117503   2048000  1000M Windows recovery environment


Disk /dev/nvme0n1: 465.76 GiB, 500107862016 bytes, 976773168 sectors
Disk model: CT500P2SSD8                             
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 00B5F5A9-82C7-434B-846B-0BD2243F1137

Device             Start       End   Sectors   Size Type
/dev/nvme0n1p1      4096   1023998   1019903   498M EFI System
/dev/nvme0n1p2   1024000   9412606   8388607     4G Microsoft basic data
/dev/nvme0n1p3   9412608 968380462 958967855 457.3G Linux filesystem
/dev/nvme0n1p4 968380464 976769070   8388607     4G Linux swap


Disk /dev/mapper/cryptswap: 4 GiB, 4294442496 bytes, 8387583 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 29.72 GiB, 31914983424 bytes, 62333952 sectors
Disk model: SD  Transcend   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 4E39CA32-5E71-B640-93C7-8F1F7505F649

Device     Start      End  Sectors  Size Type
/dev/sda1   2048 62333918 62331871 29.7G Linux filesystem
```

In this case, the SD card is recognized as `/dev/sda`. Make absolutely sure you're accessing the right device! Now you can unzip and flash:

```
$ unzip <compressed_image>.zip
$ sudo umount <device_location>
$ sudo dd bs=4M if=<raw_image>.img of=<device_location>
```

Once flashed, you can take the SD card out and put in into the Jetson Nano. I did the intial setup with a keyboard, mouse and monitor hooked up to the board. I called my user account tbilik, so that's the user account listed in the documentation. I did most development in headless mode. You can do this by connecting the micro USB on the Jetson to your computer, and establishing an SSH connection:

```
ssh tbilik@192.168.55.1
```

TRAMP mode in GNU/Emacs also works great! I did most coding and file management with GNU/Emacs for my project. [Here's a guide for setting up TRAMP mode.](https://www.emacswiki.org/emacs/TrampMode)

![Dev Setup](dev-setup.png)

## My Object Detection Model

There are several frameworks for machine learning object detection algorithms. Region proposal based frameworks and regression/classification based frameworks are the two main categories of frameworks. Region proposal based frameworks have a more general pipeline in the beginning of the section: the framework generates various image windows to test, and then it classifies the image windows. The most popular framework in the category is the R-CNN framework. Given an image, the R-CNN generates approximately 2,000 image windows to test using a selective search algorithm. It warps and re-scales the image windows with objects detected to fit in a 227 by 227 image. The re-scaling is a requirement for the neural network that classifies the windows. The selective search algorithm can produce redundant windows (two windows that contain the same object), and the rescaling process can be time-consuming. Other frameworks, such as Fast R-CNN and Faster R-CNN, use different search algorithms along with classifiers that allow for multiple aspect ratios, but otherwise work similarly to R-CNN.

Regression/classification based frameworks don’t use the general pipeline that region proposal based frameworks do and are informally known as “one-step” frameworks. YOLO and SSD are the most famous frameworks in this category. YOLO, developed by Joseph Redmon, applies a fixed grid onto an image. YOLO then uses a convolutional neural network to determine whether a gridbox contains an object with a reported level of confidence. It also generates class-specific confidence scores. With the confidence map, it then generates the bounds of the region. SSD, developed by Wei Liu, works in a similar manner to YOLO, but allows for multiple variable grids instead of a single fixed grid. This improves both speed and accuracy, and is therefore preferred over YOLO in most applications. Both frameworks struggle with detecting small objects compared to region proposal based frameworks, but SSD and YOLO can accomplish higher speeds. When doing on-the-fly object detection in live video feeds, regression/classification based frameworks are often the better choice.

For my project, I used the SSD framework, and trained it on the [traffic sign data set from the Open Images Dataset V6](https://storage.googleapis.com/openimages/web/visualizer/index.html?set=train&type=detection&c=%2Fm%2F01mqdt). The training process is detailed below, but my trained model can be found in my [jetson inference repository](https://github.com/tbilik/jetson-inference). Here are some example test images.

![Object Detection Example 1](ssd-1.png)
![Object Detection Example 2](ssd-2.png)
![Object Detection Example 3](ssd-3.png)

## Object Detection Model Training

The first necessary to training the model is to download and compile the jetson inference libraries. These libraries allow for the usage of classification models, object detection models, and semantic segmentation models in Python/C++. [Nvidia has a good guide on how to set this up](https://github.com/dusty-nv/jetson-inference/blob/master/docs/building-repo-2.md), so I'm not going to delve into many details in this documentation.

When you're at the "download models" screen of the build process, make sure to select "SSD-Mobilenet-V2" from the object detection models list.

![Object Detection Models](model-downloader.png)

Make sure to install PyTorch too! This will be necessary for training.

![PyTorch](pytorch-install.png)

## Classification Model (Attempt)

## OCR Investigations

## Setting up Tesseract on the Jetson Nano

## Developing a Power Supply for a Vehicle

I couldn't find a convenient power supply on the market that could supply the power needed from a vehicle to the Jetson Nano. Most USB power supplies for vehicles don't output enough power. Whether you're tapping into the cigarette lighter port, the fuse box, or the radio head unit, it will output approximately 12 Volts, plus or minus 2 Volts. I decided to tap into the power from the cigarette lighter plug, as it's the most convenient. In my car, the fusebox is located near the door hinge, so it's an inconvenient power source. I used an adapter that broke out the power and ground connections from the cigarrete lighter plug, and soldered that to a buck converter that steps down the power to 5 Volts. It outputs a maximum of 3 Amps, which seems to be enough in my case. If you're hooking up more peripherals or equipment to the Jetson Nano, you may need a 4 Amp buck converter.

For non-USB power, the Jetson Nano can be powered with the barrel jack or with the power and ground connections on the GPIO layout. Using the barrel jack looks cleaner aesthetically, so I opted to do that. I used a cable that took the power and ground connections from the buck converter and outputed the power through a 5.5x2.5mm barrel jack.

The wire-to-wire soldering is relatively straightforward. Solder the red and black wires from the cigarette light plug adapter to the red and black wires of the buck converter input. Solder the red and black wires of the barrel jack cable to the yellow and black wires of the buck converter output. Make sure you're using heatshrinks to cover up the solder joints, assuming you don't want to short the car battery! The end result should look something like this:

## Enabling PWM on the Jetson Nano

Pulse Width Modulation (PWM) creates a [square wave digital signal](https://www.analogictips.com/pulse-width-modulation-pwm/). This allows for controlling brightness of an LED, changing the frequency of a passive buzzer, etc. The Jetson Nano doesn't have any software PWM and only has hardware PWM on two pins. Hardware PWM isn't enabled by default, though. Annoyingly, the hardware PWM will quietly not work in its disabled state. It doesn't throw an error, permission denied, etc. It initially made me think there was an issue with my carrier board when I tried to use it and it didn't work! Anyways, to enable it, run the following:

```
sudo /opt/nvidia/jetson-io/jetson-io.py
```

Select "configure Jetson for compatible hardware" in the command line menu. You should be brought to a screen that looks like this:

![Jetson Expansion Screen](jetson-expansion.png)

Enable pwm0 and pwm2 in this menu. You can then hit "back" on the menu, and then select "save and reboot to reconfigure pins". Once rebooted, PWM should be good to go! If you want to do some tests with LEDs, [Paul McWhorter has a good video explaining this all.](https://www.youtube.com/watch?v=eImDQ0PVu2Y)

## OBD-II: Preparation

## OBD-II: Connection

## Getting the code

## Setting up a cronjob

## Project Improvements

## Thanks!

![Ron Paul Gif](ronpaul.gif)
