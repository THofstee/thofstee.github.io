+++
author = ""
comments = true
date = "2017-03-04T03:53:45-05:00"
description = ""
draft = false
featured = false
image = ""
menu = ""
share = true
slug = "zedboard-linux"
tags = ["tag1", "tag2"]
title = "zedboard_setup_linux"

+++

First, we need to figure out where out ZedBoard is connected.

At the command line, enter the following: `dmesg | grep tty`

You should see an output like this:

    [    0.000000] console [tty0] enabled
    [ 8085.395587] cdc_acm 3-1:1.0: ttyACM0: USB ACM device
    [ 8202.719767] cdc_acm 3-1:1.0: ttyACM0: USB ACM device

This tells us that the ZedBoard is on the tty port ttyACM0

--

Now we need to communicate with it. To do this we'll need a terminal that supports serial communication. A simple one to use is `minicom`. (`sudo apt-get install minicom`)

Once installed, launch minicom. You should see a screen that looks like this:

    Welcome to minicom 2.7

    OPTIONS: I18n 
    Compiled on Feb  7 2016, 13:37:27.
    Port /dev/ttyACM0, 16:26:08

    Press CTRL-A Z for help on special keys

If you get the following, make sure you have the ZedBoard turned on before launching minicom.

    minicom: cannot open /dev/ttyACM0: No such file or directory

First, we need to configure minicom. `Ctrl-A Z` will show you a list of commands available. From this page you can press `O` to get to the configuration options, or you can just type `Ctrl+A O` instead.

We'll need to modify two things, both related to the serial port. Select the `Serial port setup` option in the menu, and press `Enter`. We need to modify the Serial Device, as well as the Baud rate. Earlier we found that the ZedBoard was in `/dev/ttyACM0`, so we'll modify the Serial Device to match. In minicom, this is option A.

For the Baud rate, we want to set it to 115200/8/n/1/n, to match the ZedBoard specs. In minicom, this is option E. You want the Current parameter at the top of this menu to say `115200 8N1`, and you can modify this by either using A and B for <next> and <prev>, and other commands respectively for Parity and Data, or you can just use the common shortcuts and press `E` to set the rate to 115200, then  `Q` for the Data, Parity, and Stopbits, to quickly get it set up correctly.

The Getting Started Guide also says to turn Hardware Flow Contnrol to OFF, but I'm not sure if that means it is not supported on the ZedBoard. If it is supported on the ZedBoard, another possible strategy for communication would be to increase the baud rate, and rely on hardware flow control to sort out the rest. There are also some transmit delay parameters (10 msec/char, 100 msec/line) to set, but I'm not sure how to set these in minicom.

http://www.tldp.org/HOWTO/Text-Terminal-HOWTO-11.html

--

Once done with this, we'll need to restart the ZedBoard. If everything is set up correctly, this will take you to a minimal environment with the ZedBoard. You can type in `help` at this prompt to see some of the possible options. You can also enter `boot` to boot into the Linux image from the SD card.

When you want to turn off the zedboard, you should enter the command `poweroff` from the booted Linux image. This will perform a shutdown sequence to prolong the life of the OLED display.

--

http://hintshop.ludvig.co.nz/show/persistent-names-usb-serial-devices/

It will usually be in one of these /dev/ttyACM* ports, but we can also create a persistent name for the zedboard by using UDEV.

First, we should figure out exactly which usb device this is, and find a way of uniquely identifying it. We can list all the available usb devices with `lsusb`.

    Bus 002 Device 002: ID 8087:8000 Intel Corp. 
    Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 001 Device 002: ID 8087:8008 Intel Corp. 
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
    Bus 003 Device 004: ID 1bcf:2c6e Sunplus Innovation Technology Inc. 
    Bus 003 Device 003: ID 06cb:2970 Synaptics, Inc. touchpad
    Bus 003 Device 002: ID 0bda:0129 Realtek Semiconductor Corp. RTS5129 Card Reader Controller
    Bus 003 Device 032: ID 046d:c539 Logitech, Inc. 
    Bus 003 Device 026: ID 0853:0100 Topre Corporation HHKB Professional
    Bus 003 Device 007: ID 0489:e076 Foxconn / Hon Hai 
    Bus 003 Device 012: ID 04b4:0008 Cypress Semiconductor Corp. 
    Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

Unforunately this isn't terribly helpful. Since the Cypress CY7C64225 chipset uses ACM, and ACM is included in the Linux USB-to-UART system, we know it's one of the Linux Foundation devices.

An easier way of getting this information is with udevadm. `udevadm info -a -n /dev/ttyACM0` will print out all the information we can gather from our device. Since we need a way of uniquely identifying this ZedBoard from other devices, such as other ZedBoards, we can use the serial number to distinguish it.

    udevadm info -a -n /dev/ttyACM0 | grep '{serial}' | head -n1
    ATTRS{serial}=="84710356351F"

Now we'll need to make a UDEV rule to symlink the device. Create a file called `99-usb-serial.rules` and put it in /etc/udev/rules.d. We'll want it to have the following lines:

    SUBSYSTEM=="tty", ATTRS{idVendor}=="04b4", ATTRS{idProduct}=="0008", ATTRS{serial}="84710356351F", SYMLINK+="zedboard"

The idVendor and idProduct are specifying the Cypress chipset, and the last line tells UDEV to create a symlink /dev/zedboard that points to /dev/ttyACM* where our device actually resides. What this means is regardless of how we connect the device to our computer, we'll always have a symlink that point to the right device node.

You might need to restart your system for these rules to take effect.

--

Now we have the ZedBoard set up, we'll presumably want to use it for more than just the ARM chip and a Linux environment. The easiest way to develop programmable logic for the ZedBoard is by using Xilinx Vivado, which you can download [here](https://www.xilinx.com/support/download.html). You can install the WebPACK version which is free, or you can use the certificate that came with the ZedBoard to install the Design Edition. Both are practically equivalent, and will work for the Zynq 7020 on the ZedBoard.

After downloading the Vivado setup binary, `chmod +x` the binary, and execute it. The only device we'll need to install support for is the Zynq 7000 series, so uncheck all the others if you want to save some storage space.

After downloading and installing Vivado, we'll need to add the following line (roughly) to our `~/.bashrc`:
    source /opt/Xilinx/Vivado/2016.4/settings64.sh

There are other scripts for csh as well, so just pick whichever one matches the shell you're using.

--

Creating, building, and running a simple program in Vivado.

Launch Vivado by typing in `vivado` from the command line. By default, this will launch Vivado in gui mode which is what we want for now. From the menu that opens, select "Create New Project", then choose a name and location. This is going to be an RTL project, so select RTL. We can skip through the menus until we need to select the FPGA. Here we can just change to the Boards tab and select the `ZedBoard` from the list of boards, since it should be listed by default with Vivado. Skip through the remaining few steps and create your project.

We're going to need two files before we can generate a bitstream for the ZedBoard: a top-level module, and a constraints file. The top-level module is just the design we're trying to put onto the FPGA, and the constraints file will be where we define the pin assignment of the board. For the top module, we'll just use Verilog and create a design that will turn on an LED when the switch is on. To do this create a file called `top.v`, and paste the following in the file:

    module top (
        input switch,
        output led
        );

        assign led = switch;

    endmodule

Before we can generate the bitstream, we need to tell Vivado where the switch and led values are coming from/going to. To do this we'll need to make a constraints file called `top.xdc`. Paste the following in that file:

    set_property IOSTANDARD LVCMOS33 [get_ports led]
    set_property IOSTANDARD LVCMOS25 [get_ports switch]
    set_property PACKAGE_PIN T22 [get_ports led]
    set_property PACKAGE_PIN F22 [get_ports switch]

To add the files to Vivado, go to File > Add Sources, then "Add or create design sources", and add `top.v`. After that, go back to Add Sources and then "Add or create constraints", and add `top.xdc`.

For some reason, the .xdc file provided by Digilent can cause compilation errors in Vivado, so just use the simple xdc file we created above for now. If you select the ZedBoard then add the .xdc file provided by Digilent as a constraint, it seems to work. You can grab it [here](http://zedboard.org/support/documentation/1521) if you're keen on trying. Note that if you want to use that .xdc, you will need to change your top module to the following:

    module top (
        input SW0,
        output LD0
        );

        assign LD0 = SW0;

    endmodule

NOTE: The default voltage on the ZedBoard is 2.5V, which can be adjusted by the jumper pin near the switches. The .xdc file provided by Digilent assumes 1.8V. Either move the jumper pin (while the board is off) to 1.8V, or change the lines in the bottom of the .xdc file so the `LVCMOS25` lines are uncommented, and the `LVCMOS18` lines are commented.

The next step is to create the bitstream, so in the left hand side of Vivado go down to "Generate Bitstream", and click it. Select OK, and when it complains about Synthesis and Implementation being either out of date or nonexistent, just tell Vivado to run Synthesis and bring it up to date. Once the bitstream is generated, we'll need to program the ZedBoard. Under the "Generate Bitstream" option there is "Open Hardware Manager". Click that, then click "Open Target" and "Auto Connect". Usually this will get Vivado to do the right thing and find the ZedBoard.

Vivado might have issues trying to find the device, so you might need to install the Digilent adept runtime. You can get that [here](https://reference.digilentinc.com/reference/software/adept/start).

After you've connected to the FPGA, click "Program Device", and select the FPGA we specified, and the bitstream we just created (which should be there by default). Hit "Program", and once the bar completes there should be a blue light on your ZedBoard. Flip the switch on the far right to verify this all works. The LED above that switch should turn on when you toggle the switch.

For more details about these, and other beginning tutorials about hardware design with FPGAs, there's a great tutorial [here](https://www.beyond-circuits.com/wordpress/tutorial/tutorial1/). From this point on the assumption is that you already know what you're doing with regards to hardware design, and is focused on building projects and IP from the command line. For example, a system targeting FPGAs might find the rest helpful.

--

Given a project, we can also generate a bitstream from the command line. Instead of launching Vivado normally, we'll launch in Tcl mode and work through the needed commands. To do this, call Vivado with `-mode tcl` on the command line. We're going to start by using the project we created in the previous part, so we'll need to load it into Vivado. To do this we use the `open_project` command, then pass in the path to the .xpr file which is the Vivado project.
    open_project hello.xpr

We need to create a Synthesis run, Implementation run, and a Bitstream. Since we already set up everything before, this is fairly straightforward. We use the command `create_run` to create these runs, which takes a `-flow` argument which is specific to the type of run and the tool we are using. When we make the Implementation run, we also need to tell Vivado that the Implementation run depends on the results from the prior Synthesis run. We do this by specifying the `-parent_run` argument. In the following code, synth and impl are the names for our runs. We'll also need to tell Vivado which stages we want it to run for Implementation. Since we want to generate a bitstream, we'll enable all the required stages. Lastly, we need to launch the runs with `launch_runs`.
    create_run -flow {Vivado Synthesis 2016} synth
    create_run -flow {Vivado Implementation 2016} -parent_run synth impl
    set_property flow {Vivado Implementation 2016} impl
    set_property STEPS.POWER_OPT_DESIGN.IS_ENABLED true [get_runs impl]
    set_property STEPS.POST_PLACE_POWER_OPT_DESIGN.IS_ENABLED true [get_runs impl]
    set_property STEPS.PHYS_OPT_DESIGN.IS_ENABLED true [get_runs impl]
    launch_runs synth impl

Full disclaimer, this above section is untested and definitely doesn't work.

--

Vivado also allows you to go through these steps without a project, called "Non-Project Mode". In Non-Project Mode, all the design files and designs happen in memory, and are not saved when exiting Vivado, unlike a project. What this means is for Non-Project Mode, you will want to make sure you have design checkpoints at major stages of the design workflow. Since this design is pretty small, we'll skip that for now.

Once again, launch Vivado in Tcl mode. This time, we'll be loading source files instead of the project. For convenience's sake, I've copied over the .xdc and .v files to a new working directory. Since we're working with verilog, we'll need to use the `read_verilog` Tcl command. We'll also want to read in our design constraints.
    read_verilog top.v
    read_xdc top.xdc

Now we need to run synthesis. We do this with the `synth_design` command. We'll need to specify our top module (in this case, it's "top"), as well as the part (for the ZedBoard, this is xc7z020clg484-1).
    synth_design -top top -part xc7z020clg484-1

Now we want to optimize, place, and route the design. This is fairly straightforward:
    opt_design
    place_design
    phys_opt_design
    route_design

Lastly, we want to generate a bitstream.
    write_bitstream top.bit

Much simpler than project mode from the command line.

--

Now that we have a bitstream, we want to flash it to the FPGA.
    # Start the labtools system
    open_hw

    # Connect to the JTAG cable
    connect_hw_server -url localhost:3121
    current_hw_target [get_hw_targets]
    open_hw_target

    # Select our FPGA
    current_hw_device [get_hw_devices xc*]
    refresh_hw_device [get_hw_devices xc*]

    # Select the bitstream
    set_property PROGRAM.FILE {top.bit} [get_hw_devices xc*]

    # Program and Refresh the FPGA
    program_hw_devices [get_hw_devices xc*]
    refresh_hw_device [get_hw_devices xc*]

--

A more advanced project, using the Zynq fabric.

cover how to use IP in non-project mode.

cover the details about arguments in synth_design

write_bitstream design.bit
write_debug_probes design.ltx

--

