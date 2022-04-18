# kvp-storage
1. [Objectives](#objectives)
2. [Getting started](#getting-started)
3. [Thought process](#thought-process)
4. [Testing and Performance](#testing-and-performance)

## Objectives
### Application
:heavy_check_mark: Your application must be developed in C or modern C++ (any version), C++ is preferred</br>
:heavy_check_mark: You are free to use any libraries to help you as you see fit, but modern C++ is quite powerful</br>
:heavy_check_mark: Your application must persistently store an arbitrary number of key-value pairs to disk</br>
:heavy_check_mark: Your application must be controlled via / (command line)stdin stdout</br>
:heavy_check_mark: Your application must allow for it to be changed to be controlled over other interface, e.g. IPC</br>
:heavy_check_mark: Your application must support SET/GET/DELETE commands</br>
:heavy_check_mark: Your application must compile and run on desktop Linux</br>

### Build
:heavy_check_mark: Your application must be added as a package to an embedded Linux build for any ARM architecture (so part of the rootfs produced by
your build system). It should run in an emulated environment such as QEMU.

### To improve
:negative_squared_cross_mark: Extend tests to cover more than just the core application, especially IpcIO and FileIO classes</br>
:negative_squared_cross_mark: Extend tests to cover high frequency messages (IPC usleep() should be fine tuned)</br>
:negative_squared_cross_mark: Monitor CPU usage</br>
:negative_squared_cross_mark: Switch to different storage method for faster and more compact storage</br>
:negative_squared_cross_mark: Add custom logger (maybe syslog)</br>
:negative_squared_cross_mark: Make main less of a mess

## Getting started
### Running application on Linux
Clone repo and cd into the directory, then run
```
./build
```
The build will generate 2 different binary versions and run tests/benchmarks against both:
```
/clay-KVPStorage/artifacts/Release/src/KVPStorage
/clay-KVPStorage/artifacts/Debug/src/KVPStorage
```
The binary will use STDIO by default but the interface method can also be specified. For help run:
```
KVPStorage -h
```
Unless otherwise specified, KVPStorage will create a folder `kvps_variables` in `/var/lib/` to store the key-value pairs. When the `ipc` method is used instead of `stdio`, KVPStorage will create a folder `fifos` in `/var/lib/`. This is mostly intended for the embedded system so I would recommend running KVPStorage with different parameters if running in your work environment.<br/>
Example of how to run KVPStorage in the current directory for both input methods:
```
KVPStorage -d .
KVPStorage -i ipc -d . -f .
```
### Buildroot
I struggled to configure Yocto to use local files instead of a remote repo so I ended up switching to Buildroot. I suspect there might be a bug in the version of Yocto I used.<br/>
For buildroot:
```
git clone git://git.buildroot.net/buildroot
cd buildroot/
make qemu_arm_versatile_defconfig
make menuconfig
```
Change the following
```
Tooolchain > C library > glibc
Tooolchain > GCC compiler version gcc 11.x
Tooolchain > Enable c++ support
```
Also add the following to package/Config.in
```
menu "Misc"
    source "package/kvps/Config.in"
endmenu
```
Then you should be able to copy the package from the provided buildroot folder over to the packages from buildroot you just created.
From inside the `buildroot` dir run:
```
echo 'BR2_PACKAGE_KVPS=y' >> .config
make
```
And then to run the image in qemu:
```
qemu-system-arm -M versatilepb -kernel output/images/zImage -dtb output/images/versatile-pb.dtb -drive file=output/images/rootfs.ext2,if=scsi -append "root=/dev/sda console=ttyAMA0,115200" -nographic
```
Note that I have named the binary `kvps` in the image instead of KVPStorage to match the style of other binaries.</br>
I have also provided a built image in buildroot-image
## Thought process
My initial thought is that existing environment variables are exactly for this purpose and would be an easy way to implement this functionality since most of the work is already done. setenv(), putenv() and unsetenv() would provide the 3 core functions. Depending on the number of variables that need to be stored, execve() could potentially be used to extend the default limit for environment variables.<br/>
#### Pros:
* Variables can easily be accessed from other applications
* Does hard work for you
* No need to specify or rely on specific file locations<br/>
#### Cons:
* Security
* Not easy to distinguish between variables belonging to our application and other applications

Store each key-value pair in its own file.<br/>
#### Pros:
* Key-value pairs are easy to access from scripts or other applications without any additional requirements
* Separation between our key-value pairs and environment variables
* Relatively easy to implement<br/>
#### Cons:
* Constantly opening and closing files is slow 
* Would take up more disk space than something like JSON, XML, YAML
* If speed is an issue, files would need to be stored in some kind of tree structure but this adds complexity which eliminates the advantages of this method

JSON, XML or YAML.<br/>
#### Pros:
* Fast
* More compact than OPTION 2
#### Cons:
* Might require additional non-standard libraries unless we write our own parser

I am not too familiar with mmap() and decided not to investigate it due to time concerns but I'm making a note to read up on it.

## Implementation
I decided to implement the individual file key-value pair method because I believe it would be a better option for a reasonable set of key-value pairs. The ability to easily read values from any other application seems like a valuable advantage.

Depending on the use case this option might not be compact or small enough. I would probably use JSON as my alternative. 

## Testing and Performance
Tests performed on my PC
* It takes ~700us to read 100 values in a set of 1000 key-value pairs
* It takes ~760us to read 100 values in a set of 10000 key-value pairs
* It takes ~800us to read 100 values in a set of 100000 key-value pairs

I was expecting a more significant difference between 1000 and 100000 key-value pairs. However, I have observed that the read time for 100000 key-value pairs is significantly slower when the device is under high disk usage already. Around 10 times slower from my observations.
Example of test output:</br>
![image](https://user-images.githubusercontent.com/17459470/162827555-83191618-2096-485f-bc1a-d3c818d1d33c.png)
