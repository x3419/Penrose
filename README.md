

# Penrose 
![alt text](RPCExploit/penrose.png "Logo Title Text 1")

Penrose is a research tool that aims to build pattern matching around common vulnerable indicators that can be identified using Procmon. I'm a total noob at hunting for LPEs so I wanted a way to get very specific about what this behavior looks like and automate the vuln hunting process.

The procmon filter being used in this project is:

`Integrity|is|medium|Exclude` - We want high or system

`Detail|contains|Impersonating: desktop|Exclude` - My user is in the desktop-asdf group so we don't care about those

All detections expect the filter to have those settings

## Detections

### 1) File Write
http://sandboxescaper.blogspot.com/2019/12/chasing-polar-bears-part-one.html

This blog goes through an example of a file write vulnerable indicator that I have implemented a check for. The gist is that there's an msi file in C:\Windows\Installer that is authored by VMWare. When you run `msiexe /fa` on msi files, interesting stuff is supposed to happen.

Lets see if we can detect this activity.

To use Penrose, you need to determine how you're going to make noise for procmon. We'll use the "simple" implementation commented in main and replace the example code with:

`msiexec /fa C:\Windows\Installer\whatever.msi`

When the code runs, procmon will record the behavior caused by the command line. Those results will then be saved, converted into csv, and then checked against the following logic:

`if there's a CreateFile NAME NOT FOUND and CreateFile with delete permission and SetRenameInformationFile near each other for the same path and BUILTIN\Users has write access to the folder then this is likely vulnerable`

Running that results in many potential file write vulns being identified including the one chosen in that blog post, *stop-listener.bat,* as we would expect.

![alt text](RPCExploit/fileModify.PNG "Title")

Now that we have findings, we can open the respective csv/pmf files to investigate what exactly was detected by Penrose.


### 2) Arbitrary File Write
I've implemented a check for arbitrary file write vulnerabilities using the following logic:

`if BUILTIN\Users has full control over a folder in which there is a CreateFile SUCCESS to a file then this can be leveraged as an arbitrary file write using a hard link`

To test this detection, we can install the unpatched Goverlan Reach Agent service binary.

Then lets replace the "make noise" code with  with:

`sc start goverlanagent`

When we kick that off, we're greeted with:

![alt text](RPCExploit/arbitraryFileWrite.png "Title")

Indicating that the detection does work as expected.




### 3) Executable code control
Just like here: https://aspe1337.blogspot.com/2017/04/writeup-of-cve-2017-7199.html

Implemented as:

`if BUILTIN\Users has write access to a path in which there is a CreateFile NO SUCH FILE where Path=*.dll or Path=*.exe then this demonstrates executable code control`

Additionally:

`if there's ever a Load Image to a directory BUILTIN\Users has full control of then that's a vulnerability`

## Example
![alt text](RPCExploit/mainExample.png "Title")

This is an example implementation of main which showcases how simple it is to set up this tool. 

In this example, we're running `msiexec` with the `/fa` repair flag on every msi file in `C:\Windows\Installer`.

## Adding fuzzing and automation
This project is meant to be agnostic of the method that you're using to make "noise" for procmon. You could make it do COM stuff, run `msiexec /fa` on every msi, whatever you want. Then if one of those iterations causes vulnerable behavior, you will know.

https://googleprojectzero.blogspot.com/2019/12/calling-local-windows-rpc-servers-from.html

James Forshaw recently released a library that can be used to call local RPC servers from .NET. This seems like a cool idea because you can do crazy stuff in .NET like reflection. Basically his blog post walks you through generating RPC server C# definitions that you can import and use. At some point I want to see if I can import _all_ RPC servers into the visual studio project, then use reflection to invoke all "fuzzable" functions (that take params like string, int, etc) and detect their vulnerable behavior.
