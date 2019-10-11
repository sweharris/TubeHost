The latest published version of this code is always at
  http://sweh.spuddy.org/Beeb/TubeHost/

There is a working GIT repo (which may be ahead of the published version)
  https://github.com/sweharris/TubeHost

This is a Unix TubeHost filesystem written in perl based around JGH's
HostFS concept

  http://mdfs.net/Software/Tube/Serial/
  http://mdfs.net/Software/Tube/Serial/HostFS.txt

In this model, the HostFS ROM makes "Tube" like communication but using
the RS423 serial port instead.  Since RS423 is a single channel in each
directions, commands are embedded in the datastream.  Essentially the
HostFS passes standard BBC calls (eg OSBGET, OSARGS, OSFSC) and data
structures across the channel, and all the complicated processing is
performed on the remote server (aka TubeHost or Host, for short).

Now this design (whilst slightly idiosyncratic - eg in how file data is
transferred) is very clever and flexible.  Ultimately it's up to the Host
as to how to present a filesystem.  *COMMANDS are also interpreted by
the Host.  In JGH's reference code he has a BASIC TubeHost server (and a
precompiled Windows .exe from it).  This has an ADFS look and feel; it's
a hierarchical filesystem, with *DIR enter subdirectories, and "." as the
directory separator and so on.  His code translates Host<->Beeb filenames
and presents them in a form native to the Beeb.

I took a more DFS-like approach, taking ideas from MMB structures.  In
my system we have a single level tree (eg $HOME/Beeb_Disks).  Each Unix
subdirectory becomes an available disk which can be "inserted" into a
drive (using the *DIN) command.  There are 10 drives available (0->9).

If the Unix directory is numbered then we assign this disk to a slot of
that number; any unnumbered directories are assigned sequential numbers

So, for example, on Unix:
    % ls Beeb_Disks
    0.Mr_Ee  1.test1  10.test10  TAPES

We would see this on the Beeb:
    >*DCAT
    
    Disks available:
         0: 0.Mr_Ee
         1: 1.test1
        10: 10.test10
        11: TAPES

"TAPES" was assigned "11" because it was the next free slot.

The rationale behind this structure was to make SSDs easier to handle.
Using MMB_utils ( http://sweh.spuddy.org/Beeb/mmb_utils.html ) we could
extract the contents of an SSD into a directory and now it shows up on
the Beeb exactly as it should.

    % cd Beeb_Disks
    % beeb getfile ~/Cylon_Attack.ssd Cylon_Attack
    Saving $.Ca as Ca
    Saving $.Cylon as Cylon
    Saving $.!BOOT as !BOOT
    % ls Cylon_Attack
    !BOOT  !BOOT.inf  Ca  Ca.inf  Cylon  Cylon.inf

And now:
    >*DCAT
    Disks available:
         0: 0.Mr_Ee
         1: 1.test1
        10: 10.test10
        11: Cylon_Attack
        12: TAPES
    >*DIN 5 Cylon_Attack
    >REM could also have done *DIN 5 11 'cos the assigned number was 11
    >*I. :5.*.*
    $.Cylon     L  FF1900 FF8023 00237E
    $.Ca        L  001100 00257D 004900
    $.!BOOT        000000 000000 000025
    >*DRIVE 5
    >*TYPE !BOOT
    10CLOSE#0
    20*FX21
    30CHAIN"CYLON"
    RUN
    >

We can see the load/exec/locked attributes have been mainted, due to the
INF files (which should be bbcim compatible).

When TubeHost starts up, any disks with numbers 0->9 are pre-loaded into
slots 0->9.  You can see the current disks loaded:
    >*HSTATUS D
    Disk 0: 0.Mr_Ee
    Disk 1: 1.test1
    Disk 2:
    Disk 3:
    Disk 4:
    Disk 5: Cylon_Attack
    Disk 6:
    Disk 7:
    Disk 8:
    Disk 9:

    Current drive: 5
    Current directory: $ (:5:$)
    Current library: :0.$
    Current folder:

There is an invisible drive L which is mapped to the _Library directory on the
host.  This will be searched for bad commands if the normal search mechanism
fails.  You can save executables here.
eg
   10FOR A=0 TO 3 STEP 3
   20P%=&900
   30[OPT A
   40LDX #0
   50.LP
   60LDA MSG,X
   70BEQ FIN
   80JSR &FFEE
   90INX
  100BNE LP
  110.FIN
  120RTS
  130.MSG
  140EQUS "HELLO"
  150EQUB 13
  160EQUB 10
  170BRK
  180]
  190NEXT
  200OSCLI "SAVE :L.GRONK 900 "+STR$~P%

Now "*GRONK" will print "HELLO".  This allows you to build out a library
of commands that will always be present.  (You can, of course, use *DIN to
change where the L drive points).

Folders
=======
In v0.13 a new concept was added, called Folders.  These are directories
in the tree that begin with the underscore (_) character.  These don't
show as disks for *DCAT/*DIN purposes, but allow us to organise disks
in an easier manner.  So, for example, you could put games disks in the
_GAMES folder, test work in the _TEST folder.

e.g.
    % find Beeb_Disks -type d | sort
    Beeb_Disks
    Beeb_Disks/_GAMES
    Beeb_Disks/_GAMES/0.Mr_Ee
    Beeb_Disks/_GAMES/1.test1
    Beeb_Disks/_GAMES/10.test10
    Beeb_Disks/_GAMES/TAPES
    Beeb_Disks/_JUNK
    Beeb_Disks/_JUNK/_FOO
    Beeb_Disks/_JUNK/_FOO/_BAR
    Beeb_Disks/_TEST
    Beeb_Disks/_TEST/Folders
    Beeb_Disks/_TEST/Stuff

Folders are displayed with *HFOLDERS, changed into with *HCF and can
be created with *HMKF.  When a disk is loaded from a folder then the
path is shown in the *HSTATUS command

    >*HFOLDERS
    Folders present:
      ..
      _GAMES
      _JUNK
      _TEST
    >*HCF _GAMES
    Current folder is: _GAMES
    >*DCAT
    Disks available:
         0: 0.Mr_Ee
         1: 1.test1
        10: 10.test10
        11: TAPES
    >*DIN 0 0
    >*HCF ..
    Current folder is: 
    >*HCF _JUNK
    Current folder is: _JUNK
    >*HCF _FOO
    Current folder is: _JUNK/_FOO
    >*HCF ^
    Current folder is: 
    >*HCF _TEST
    Current folder is: _TEST
    >*DIN 1 Folders
    >*HSTATUS D
    Disk 0: _GAMES/0.Mr_Ee
    Disk 1: _TEST/Folders
    Disk 2: 
    Disk 3: 
    Disk 4: 
    Disk 5: 
    Disk 6: 
    Disk 7: 
    Disk 8: 
    Disk 9: 
    
    Current drive: 0
    Current directory: $ (:0.$)
    Current library: :0.$
    Current folder: _TEST


Functions handled
=================

  0x0C => 'OSARGS',
    Y=0, A=0 return current filesystem in A  (Also handled in HOSTFS ROM)
    Y=0, A=1 return rest of command line   (Handled in HOSTFS ROM)
    Y=0, A=255 sync all open files
   Y!=0, A=0 Get PTR#
   Y!=0, A=1 Set PTR#
   Y!=0, A=2 Read length (EXT#)
   Y!=0, A=3 Set length (EXT#)   [ see note ]
   Y!=0, A=255 Sync this file

  0x0E => 'OSBGET',
    Y=filehandle, read byte from file into A, C set if EOF before read occured

  0x10 => 'OSBPUT',
    Y=filehandle, write byte from A into file

  0x12 => 'OSFIND',
    A=0, close file Y
    A=40, openin 
    A=80, openout
    A=C0, openup

  0x14 => 'OSFILE',
    00 SAVE
    01 rewrite load/exec/attr
    02 rewrite load
    03 rewrite exec
    04 rewrite attr
    05 read catalog entry
    06 Delete
    07 Create empty file of defined size
    FF LOAD

  0x16 => 'OSGBPB',
    A=1 write bytes to new ptr
    A=2 write bytes to current ptr
    A=3 get bytes from new ptr
    A=4 get bytes from current ptr
    A=5 Read disks title, boot opts
    A=6 read directory name
    A=7 read library name
    A=8 read files from directory

  0x18 => 'OSFSC',
    00 *OPT  [ see note ]
    01 CheckEOF
    02 */
    03 *command
    04 *RUN
    05 *CAT
    06 New filesystem  (handled by HOSTFS ROM)
    07 Handles (handled by HOSTFS ROM?  Done here as 0x80-0x9F just in case)
    08 *command alert  (dropped by HOSTFS ROM)
    09 *EX
    0A *INFO
    0B *RUN for library  [ see note ]
    FF called by HostFS ROM at shift-boot time [ see note ]
    
COMMENTS
=========
OSARGS Y!=0, A=3 Set length (EXT#) - Neither B nor Master support this;
  the code is there but I dunno if it'll work

OSFILE 07 Create empty file of defined size - appears to work.

OSFSC
  00 *OPT - "*OPT 5" sets debug level on host.
            "*OPT 6" sets speed delays (see below)
            No other *OPT value does anything
  0B *RUN for library - not sure how to verify this one works!

OSGBPB
  A=5 Read disks title, boot opts - disk name truncated to 12 chars if
       necessary
  A=8 read files from directory - This appears to work.  Note that Unix
    filenames may be long and BBC programs may expect 7 or 10 character
    limits; not a bug in TubeHost but in Beeb programs

A full Tube Host should also handle other types of calls (eg OSRDCHIO).
These are not handled by this program; we are a pure filesystem server
and not a generic Host application.  Calls *NOT* handled:
   0x00 => 'OSRDCHIO',
   0x02 => 'OSCLI',
   0x04 => 'OSBYTELO',
   0x06 => 'OSBYTEHI',
   0x08 => 'OSWORD',
   0x0A => 'OSWORD0',

*COMMANDS
=========
can be prefixed with a ! if a ROM or OS commands conflicts
  *HCF
     Host Change Folder
     Changes the folder we look in for disks.  Folder names must start
     with an underscore (_).  A special name of ".." means go up a level,
     and "^" means go to the top of the tree

  *HFOLDERS
     Show sub-folders under the current folder.  Folders can be nested
     as deeply as the filename allows.

  *HMKF
     Host Make Folder
     Creates a new subfolder in the current folder.  Folder names must
     start with an underscore (_)

  *HSTATUS [DF]
     Displays status of Host, including what disks are in what slots,
     current directory/library status, and what files are currently open.
     "D" limits to disks/directories; "F" limits to open files
  
  *DCAT
     Shows what disks (Unix subdirectories) are available
   
  *DCREATE diskname
     Creates a new Unix subdirectory
   
  *DIN drive disk
     Inserts "disk" into drive
   
  *DBOOT disk
     Inserts "disk" into drive 0 and then sends a small program to &0900
     which does a *FX 255,7,0 and then calls (&FFFC).  The result of this
     should be the same as inserting the disk and then pressing shift-break

  *DOUT drive
     Ejects "disk" from drive
   
  *DRIVE drive
  *DIR dir
  *LIB dir
     As per DFS
    
  *HRESET
     Causes TubeHost to re-exec itself, which effectively resets it
     to startup mode (default disks, default directories, no open files etc)
   
  *IAM
  *I AM
     Identify yourself.  So far this has no use except for reporting in
     *VERSION but might be usable in the future

  *VERSION
     returns the version of the TubeHost code being run

  *INFO filespec
  *DELETE file
  *RENAME file1 file2
  *ACCESS file [L]
     As per DFS

  *DMP file
     Test emulation of *DUMP performed on Host side
   
  *FOO count
     Debug command; sends "count" number of X's to the screen

Running
=======

This is pure perl code.  To talk to a serial port you need the
Device::SerialPort module.  I'm not sure if I'm making any hard assumptions
around Unix/Linux so this may work on other platforms.

./TubeHost [-s|-u] device [speed]

   "-s" ==> serial device (default)
   "-u" ==> Unix domain socket
   "-U" ==> UPURS device (/dev/ttyUSB0 at 115200).
   device ==> filename to connect to
   speed ==> Baud rate (only for serial; 9600 default).

"Unix domain socket" is for VirtualBox users and BeebEm.  It's possible
to configure VirtualBox so the client OS (Windows, for example) has its
serial port mapped to a Unix domain socket.  Then BeebEm can connect its
serial port to the client OS serial port.  In this way TubeHost can talk
to HostFS inside BeebEm inside Virtualbox.  It's how I did my debugging!

"UPURS" is for the "UPURSFS" ROM which uses MartinB's clever UPURS
cable (effectively emulating a serial port via the user port) which
runs at 115200 baud.  Only tested to work with FTDI based USB serial
adapters.

*OPT6
=====
Now HostFS and TubeHost is totally reliant on a clean serial channel.
I've found, in testing, that sending data too quickly can cause
characters to be lost, and thus corruption.  I've seen this with both
real serial ports and the VirtualBox emulated serial port.  It's unclear
if the issue is in HostFS or in my serial cable and the Linux/Virtualbox
serial emulation.  To work around this the serial send routines
have delays built into them; after "A" characters have been sent, delay
for "B" microseconds.  The default is "A=1,B=2", which causes a 1ms
delay after each character sent.  Throughput on a 9600 baud connection
works out around 720cps.  Not as fast as floppy, but a lot faster than
tape :-)  With UPURS the default values are A=0,B=0.

These values can be tuned with "*OPT6".  Convert AB into hex; &AB (or
A*16+B) and pass to *OPT6.  So the default is, effectively, *OPT6 18.
If you are finding communication issues you can use *OPT6 and "*FOO"
to help tune these timings.  Once you have good values then you can, of
course, hard code them into the script.

BREAK/SHIFT-BREAK/CONTROL-BREAK
===============================
State is maintained on the server, so pressing BREAK doesn't change
anything.  If you press SHIFT+BREAK then all file handles are closed,
dir and lib is set to :0.$ and an indicator is sent back to the client
as of the OPT4 status of drive 0.  With a modified HostFS client this
may allow boot.   

Now, in theory, control-break should also send an update to the server,
but if the serial port is not connected or if the TubeHost software is
not running or baud rates are wrong then this causes a hang at power up.
To this end the client doesn't let the server know that control-break has
been sent.  This means TubeHost is subtly different in behaviour to disks.

===================================================================

This program is free software; you can redistribute it and/or
modify it under the terms of the GNU General Public License
as published by the Free Software Foundation; either version 2
of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program; if not, write to the Free Software
Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301, USA.
