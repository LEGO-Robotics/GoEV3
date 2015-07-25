GoEV3
=====

Introduction
------------

In 2013, LEGO introduced its third-generation Mindstorms robotics set, the [EV3](http://en.wikipedia.org/wiki/Lego_Mindstorms_EV3). Unlike its predecessors, the EV3 runs Linux, giving hackers and hobbyists the opportunity to create robots more capable than ever before.

The [ev3dev project](http://www.ev3dev.org/) maintains open source, hacker-friendly releases of EV3's operating system. Distributions include built-in [ssh](http://en.wikipedia.org/wiki/Secure_Shell) support and custom drivers for EV3's hardware. In fact, a simple file system-based interface can be used to interact with EV3's motors, sensors, buttons, and LEDs. Directories under `/sys/class` represent various device classes, and setting attributes is as simple as writing to files.

For example, executing the following shell commands will run a motor (assuming it was connected to some port first time after robot had been turned on) at 50% speed:

	echo  50 > /sys/class/tacho-motor/motor0/duty_cycle_sp
	echo run-forever > /sys/class/tacho-motor/motor0/command

This enables third-party developers to write EV3 bindings for any programming language that has a file system IO API.

GoEV3 provides EV3 bindings for [Google Go](http://golang.org) (golang), enabling Mindstorms robot programmers to take advantage of Go's efficiency, sane and clear concurrency model, modern syntax and extensive standard library.

GoEV3 was originally developed by [Matt Rajca](https://github.com/mattrajca). For now the development is going on here in this fork. 

Getting Started
---------------

### ev3dev

First, we need to install ev3dev onto a Micro SD card (by using an SD card, we can keep EV3's built-in software intact). Instructions for the installation process can be found on [ev3dev's getting started page](http://www.ev3dev.org/docs/getting-started/). The following assumes that you have chosen the latest ev3dev's release. When you're done, reboot your EV3 and make sure you can ssh into it from your computer.

### Google Go

Next, we need to install an ARMv5 build of Google Go. Fortunately for us, Go developer Dave Cheney released [builds](http://dave.cheney.net/unofficial-arm-tarballs) of Go for various ARM architectures. On your computer, download the latest ARMv5 package (1.2 as of this writing). Once the download completes, transfer it to the EV3 over ssh using [scp](http://en.wikipedia.org/wiki/Secure_copy):

	scp /path/to/go1.2.linux-arm~armv5-1.tar.gz root@192.168.3.2:~/go.tar.gz

Be sure to replace `192.168.3.2` with your EV3's IP address. Now we can ssh into the EV3 and extract the archive to its final destination:

	cd /usr/local
	tar -xf ~/go.tar.gz

Extraction may take a few minutes. Once it's done, we'll add Go's `bin` directory to our shell's path:

	echo "export PATH=\$PATH:/usr/local/go/bin" >> ~/.bashrc
	source ~/.bashrc

We should now be able to invoke the `go` tool like so:

	root@ev3dev:~# go version
	go version go1.2 linux/arm

### GoEV3

Now that we have Google Go up and running, we need to install GoEV3. First, let's set up our Go workspace:

	cd ~
	mkdir gocode
	echo "export GOPATH=\$HOME/gocode" >> ~/.bashrc
	source ~/.bashrc

We can obtain GoEV3 from its GitHub repository. Be sure to have internet connection sharing enabled prior to running the following commands:

	mkdir -p gocode/src/github.com/ldmberman
	cd gocode/src/github.com/ldmberman
	wget -O GoEV3.tar.gz --no-check-certificate https://github.com/ldmberman/GoEV3/archive/0.2.0.tar.gz
	tar -xf GoEV3.tar.gz
	mv GoEV3-0.2.0 GoEV3
	rm GoEV3.tar.gz
	cd ~

Note we're not using `go get` to avoid installing `git` on the EV3.

GoEV3 comes with a sample program that lets us exercise EV3's various hardware capabilities. We can now run it with the following commands:

	go install github.com/ldmberman/GoEV3
	gocode/bin/GoEV3

Choose mode `6. Motors`, plug in a motor to output port A, and watch it turn! Feel free to explore the other modes.

Your First Program
------------------

We'll now write a short Go program that drives a robot forward as long as there are no obstacles. To do this, we'll take advantage of the Sensors and Motor APIs.

First, let's create a new package, `example1`.

	mkdir ~/gocode/src/example1
	cd ~/gocode/src/example1
	nano main.go

Now paste in the following code:

	package main
	
	import (
		"github.com/ldmberman/GoEV3/Motor"
		"github.com/ldmberman/GoEV3/Sensors"
		"time"
	)
	
	func main() {
		sensor := Sensors.FindInfraredSensor(Sensors.InPort2)
		
		Motor.Run(Motor.OutPortA, 40)
		Motor.Run(Motor.OutPortB, 40)
		
		for {
			value := sensor.ReadProximity()
			
			if value < 50 {
				break
			}
			
			time.Sleep(time.Millisecond * 100)
		}
		
		Motor.Stop(Motor.OutPortA)
		Motor.Stop(Motor.OutPortB)
	}

This code assumes we have an infrared sensor attached to input port 2 and two motors attached to output ports A and B.

To run the program, save the file, exit nano, and execute the following command:

	go run main.go

The motors on ports A and B will start turning. To stop them, simply extend your hand in front of the infrared sensor.

If you prefer writing and compiling Go programs on your computer, you can [cross-compile](http://dave.cheney.net/2012/09/08/an-introduction-to-cross-compilation-with-go) an ARMv5 binary and transfer it to the EV3 over scp.

Auto Login + Launcher
---------------------

We can configure the EV3 to automatically log in to a program launcher at boot. The [GoEV3 Launcher](https://github.com/ldmberman/GoEV3-Launcher) will prompt us to run any Go program found in the `GOPATH/bin` directory.

We need to obtain the GoEV3 Launcher:

	cd ~/gocode/src/github.com/ldmberman
	wget -O Launcher.tar.gz --no-check-certificate https://github.com/ldmberman/GoEV3-Launcher/archive/master.tar.gz
	tar -xf Launcher.tar.gz
	mv GoEV3-Launcher-master Launcher
	rm Launcher.tar.gz
	cd ~

Let's build it and try it out:

	go install github.com/ldmberman/Launcher
	gocode/bin/Launcher

Select `GoEV3` and hit the Enter (center) button on the EV3.

Finally, let's configure the Launcher to run on login:

	nano ~/.bashrc

Add the following lines to the end of the file:

	if [ -z "$SSH_CLIENT" ]; then
		$GOPATH/bin/Launcher
	fi

The GoEV3 Launcher will now run automatically on boot. Reboot the EV3 to try it out.

Thread Safety
-------------

All function and method calls in GoEV3 are thread-safe.

Documentation
-------------

The complete documentation for GoEV3 can be found on [godoc](https://godoc.org/github.com/ldmberman/GoEV3).

Contributing
------------

GoEV3 is still in its early stages and subject to API changes as the ev3dev project evolves. Filing issues and submitting pull requests are the two best ways to get involved. Documentation improvements, new APIs, example programs, and bug fixes are all welcome.
