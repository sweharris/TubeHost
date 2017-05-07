#!/usr/bin/perl

# TubeHost
#   This is a Unix TubeHost filesystem written in perl based around JGH's
#   HostFS concept
#     http://mdfs.net/Software/Tube/Serial/
#     http://mdfs.net/Software/Tube/Serial/HostFS.txt
#
# This code lives at http://sweh.spuddy.org/Beeb/TubeHost/
#
# v0.01 - 2013/02/16 - first release 
# v0.02 - 2013/02/16 - mess around with Device::SerialPort
#                      to avoid need for delays
# v0.03 - 2013/04/06 - Minor tweaks to handle UPURS
# v0.04 - 2013/04/07 - *OPT4 handling; shift-break handling
# v0.05 - 2013/04/20 - Hide *OPT4 value from most routines.  Fix OSFILE getattr
#                      to also return length properly!
# v0.06 - 2013/09/07 - *HSTATUS and *DCREATE were sending an additional <00>
#                      which was confusing the client no end!
# v0.07 - 2013/09/15 - add a pseudo "L" drive for _Library mapping; always
#                      search this if command not found in $dir or $lib
#                      Add *IAM and *VERSION
# v0.08 - 2013/09/22 - add *DBOOT - it cheats and sends back a binary
#                      that does a *FX 255 7 and then calls (&FFFC)
#                      Allow disk names with a number prefix to be called
#                      without the number
# v0.09 - 2013/09/22 - Move IAM later down the list so *I. calls INFO

my $VERSION="0.09";

my $SERIAL=1;
my $UPURS=0;
my $socket_path="/dev/ttyS0";
my $BAUD=9600;
my $PAUSE_AFTER=1;
my $PAUSE_DELAY=2;
my $DEBUG=$ENV{DEBUG} || 0;
my $disk_base="/home/sweh/Beeb_Disks";
my $identity="Anonymous";

my @ORIG_ARGV=@ARGV;

if ($ARGV[0] eq '-s' )
{
  $SERIAL=1;
  shift @ARGV;
}
elsif ($ARGV[0] eq '-u' )
{
  $SERIAL=0; shift @ARGV;
}
elsif ($ARGV[0] eq '-U' )
{
  $SERIAL=0;
  $UPURS=1;
  $PAUSE_AFTER=0;
  $PAUSE_DELAY=0;
  $socket_path="/dev/ttyUSB0";
  $BAUD=115200;
  shift @ARGV;
}

$socket_path=$ARGV[0] if $ARGV[0];
$BAUD=$ARGV[1] if $ARGV[1];

eval "use Device::SerialPort" if $SERIAL;

use strict;
use FileHandle;
use Socket;
use IO::Socket::UNIX qw( SOCK_STREAM );

my $FILE_LOCKED=8;
my $FILE_UNLOCKED=0;

my $FH_LOW=0x80;
my $FH_HIGH=0x9f;

my %disks=();
$disks{"L"}="_Library";
my $drive=0;
my $dir='$';
my $lib=':0.$';

my %open_files=();
#  For open_files
#     index is handle; sub-keys {canonical_name} {ptr} {ext} {content} {mode} {dirty} {fullpath}
#     We pretty much hold everything in memory until close/sync

# For delays in serial writing
my $char_written=0;

# Not currently used, really
my %opt=();

my $socket;  # I/O channel

my $escape=chr(0x7F);

# We only implement a subset of commands; this isn't a full tube
# Host, just a Host to handle HostFS.  The full spec is:
#   0x00 => 'OSRDCHIO',
#   0x02 => 'OSCLI',
#   0x04 => 'OSBYTELO',
#   0x06 => 'OSBYTEHI',
#   0x08 => 'OSWORD',
#   0x0A => 'OSWORD0',
#   0x0C => 'OSARGS',
#   0x0E => 'OSBGET',
#   0x10 => 'OSBPUT',
#   0x12 => 'OSFIND',
#   0x14 => 'OSFILE',
#   0x16 => 'OSGBPB',
#   0x18 => 'OSFSC',

# What we handle:
my %commands=
(
  0x0C => 'OSARGS',
  0x0E => 'OSBGET',
  0x10 => 'OSBPUT',
  0x12 => 'OSFIND',
  0x14 => 'OSFILE',
  0x16 => 'OSGBPB',
  0x18 => 'OSFSC',
);

# ==========================================================================
# SECTION: Utility functions for string handling
# ==========================================================================

sub clean_ascii($;$)
{
  my ($msg,$plus)=@_;
  $msg=~s/([^${plus}!-~])/"<" . to_hex(ord($1)) .">"/eg;
  return $msg;
}

sub debug($$)
{
  my ($lvl,$msg)=@_;
  my $c=(caller(1))[3]; $c=~s/^.*:://; $c="main" unless $c;

  if ($lvl <= $DEBUG) { printf STDERR "%s %20s %02d: %s\n",scalar localtime(time),$c,$lvl,clean_ascii($msg," "); }
}

sub to_hex($)
{
  return sprintf("%02X",$_[0]);
}

sub to_hex8($)
{
  return sprintf("%08X",$_[0]);
}

# See if the passed command matches the request (allow "." abbrevs)
sub cmd_match($$)
{
  my ($user,$cmd)=@_;
  # Optionally the command could begin with a !
  $user=~s/^!//;
  return 1 if $user eq $cmd ||
             (substr($user,-1) eq '.' &&
              substr($user,0,-1) eq substr($cmd,0,length($user)-1)); # abbrev
  return 0;
}

sub need_arg($)
{
  my ($str)=@_;
  error(214,"Missing argument") unless $str ne "";
  return $str;
}

# ==========================================================================
# SECTION: Serial I/O routines for communication with Client
# ==========================================================================

sub raw_read_char()
{
  my ($d,$l)=("",0);

  if ($SERIAL)
  {
    while (!$l) 
    {
      ($l,$d)=$socket->read(1);
    }
  }
  else
  {
    $l=sysread($socket,$d,1,0);
    die "Socket closed!\n" unless $l;
  }

  debug(30,"Character read: $d");
  # At this point, reset char_written 'cos the other end must be ready!
  $char_written=0;
  return($d);
}

# This is also a 'dispatch' loop; if "escape_good" is set and an escape
# sequence is received then we decode the command and call the relevant
# routines.  Pretty much only MAIN should call read_char(1), but other
# routines can call read_char so as to handle the escape-escape case.
sub read_char(;$)
{
  my ($escape_good)=@_;

  my $ch="";
  while ($ch eq "")
  {
    $ch=raw_read_char;
    if ($ch eq $escape)
    {
      debug(20,"Escape character detected");
      $ch=raw_read_char;
      if ($ch ne $escape)
      {
        error(255,"Unexpected escape command received") unless $escape_good;
        my $i=ord($ch);
        debug(20,"Escape command received: &" . to_hex($i));
        my $cmd=($i & 0x1E);
        $cmd=$commands{$cmd};

        if ($cmd && defined(&$cmd))
        {
          my $fn=\&$cmd;
          debug(10,"Command: $cmd");
          $fn->();
        }
        elsif ($cmd)
        {
          debug(1,"Command $cmd received but no action defined");
        }
        else
        {
          debug(1,"Invalid command string $cmd ignored");
        }
        $ch="";
      }
    }
  }
  return($ch);
}

sub read_byte()
{
  return ord(read_char);
}

sub read_string()
{
  my $string="";
  my $c;
  while (($c=read_char) ne '')
  {
    $string .= $c unless ord($c)==13;
    last if ord($c)==13;
  }
  debug(5,"Read string: $string");
  return $string;
}

sub read_addr()
{
  my ($a,$l);
  foreach $l (0..3)
  {
    $a=$a*256+read_byte;
  }
  debug(5,"Address loaded: " . to_hex8($a));
  return $a;
}

# As the client to send us data
sub read_data_from_memory($$)
{
  my ($save,$len)=@_;
  my $data;
  send_cmd(0xf0);
  send_addr($save);
  foreach (1..$len)
  {
    $data .= read_char;
  }
  debug(1,"Receive $len characters complete");
  send_cmd(0xb0);
  return $data;
}

sub raw_write($)
{
  my($b)=@_;
  debug(70,"Sending byte: $b");

  if ($SERIAL)
  {
    $socket->write($b);
    $socket->write_drain;
  }
  else
  {
    syswrite($socket,$b);
  }
  if ($PAUSE_AFTER && $PAUSE_DELAY)
  {
    $char_written++;
    if ($char_written >= $PAUSE_AFTER)
    {
      select(undef,undef,undef,$PAUSE_DELAY/1000);
      $char_written=0;
    }
  }
}

sub send_byte($)
{
  my ($msg)=@_;
  debug(60,"Sending byte: " . to_hex($msg));
  raw_write(chr($msg));
  raw_write(chr($msg)) if (chr($msg) eq $escape);
}

sub send_bytes(@)
{
  debug(5,"Sending " . @_ . " bytes");
  foreach (@_) { send_byte($_); }
}

sub send_addr($)
{
  my ($addr)=@_;

  debug(5,"Sending address: " . to_hex8($addr));
  send_byte(($addr & 0xff000000)/0x1000000);
  send_byte(($addr & 0xff0000)/0x10000);
  send_byte(($addr & 0xff00)/0x100);
  send_byte($addr & 0xff);
}

sub send_cmd($)
{
  my ($msg)=@_;
  debug(5,"Sending command: " . to_hex($msg));
  raw_write($escape);
  raw_write(chr($msg));
}

sub send_reply($)
{
  my ($msg)=@_;
  debug(5,"Replying: $msg");
  foreach (split(//,$msg))
  {
    raw_write($_);
    raw_write($_) if ($_ eq $escape);
  }
  raw_write(chr(0));
}

sub send_output($)
{
  send_byte(0x40);
  send_reply($_[0]);
}

sub send_data_to_memory($@)
{
  my ($load,@data)=@_;
  debug(5,"Sending " . @data . " bytes to " . to_hex8($load));
  # Eat my data
  send_cmd(0xe0);    # I'm gonna send you a file
  send_addr($load);
  send_bytes(@data);
  send_cmd(0xb0);    # end of data
  debug(5,"Send data to memory complete");
}

sub send_file($$)
{
  my ($want,$load)=@_;
  debug(10,"Sending file $want to $load");
  my @data=load_file($want);
  send_data_to_memory($load,@data);
}

sub error($$)
{
  my ($err,$str)=@_;
  debug(1,"Error " . to_hex($err) . " $str");
  send_cmd(0);
  send_byte($err);
  send_reply($str);
  goto MAIN;
}

# ==========================================================================
# SECTION: File/Catalogue handling
# ==========================================================================

sub CalcCRC($)
{
  my $crc=0;

  foreach my $c (split(//,$_[0]))
  {
    $crc ^= (256*ord($c));
    foreach my $x (0..7)
    {
      $crc *= 2;
      if ($crc > 65535)
      {
        $crc-=65535; $crc ^= 0x1020;
      }
    }
  }
  return($crc);
}

sub gen_path($$)
{
  my ($drive,$n)=@_;

  # These keep Unix filenames at least a little sane
  $n=~tr/ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789_+.!@$%,-/_/c;
  $n=~s/^\$\.// unless $n eq '$.';

  if ( -e $n )
  {
    my $n1=$n;
    my $cnt=0;
    while ( -e $n1 ) { $n1 = $n . "-" . ($cnt++); }
    $n=$n1;
  }
  return $disk_base . "/" . $disks{$drive} . "/$n";
}

# A string may be
#  FILENAME
#  "FILENAME"
#  FILENAME paramters
# etc
#
# Find the last " on the line, then keep going until whitespace (or <OD>).
# That's our truncation point.  Now if there is an odd number of quotes we
# have a bad filename.  Next if the string starts and ends with "
# strip them off.  Now convert "" to ".  At this point we can then perform
# any Host-OS-specific translations, wildcard matching and so on.

sub extract_filename($)
{
  my ($str)=@_;

  # Strip off trailing CR/LF
  $str=~s/[\r\n]+$//;

  # Strip leading spaces
  $str=~s/^\s+//;

  # Find the last " (if any)
  my $s=rindex($str,'"')+1;
  $s=0 if $s<0;

  # Search forward until we get a high-bit or control character
  $str=~s/^(.{$s}[^\x0-\x20\x7f-\xff]*).*$/$1/;

  # If the first character is a " and the last character is a "
  # then strip them
  $str=~s/^"(.*)"$/$1/;

  # We should have an even number of " symbols now
  my $num = $str =~ tr/"//;
  if ($num%2) { error(253,"Bad filename: odd quotes"); }

  # Now convert "" to "
  $str=~s/""/"/g;
  debug(10,"We want file: $str");
  return($str);
}

sub check_drive($)
{
  my ($dr)=@_;
  if ($dr !~ /^\d$/ && $dr ne 'L')
  {
    error(205,"Bad drive: $dr");
  }
  return $dr;
}

sub check_disk_num($$)
{
  my ($d,$disks)=@_;
  if ($d !~ /^\d+$/)
  {
    error(199,"Bad disk number: $d");
  }
  error(199,"No disk $d") unless defined $disks->{$d};
  return $d;
}

sub check_drive_in_use($)
{
  my ($dr)=@_;
  foreach (keys %open_files)
  {
    error(194,"Open file on drive") if substr($open_files{$_}{canonical_name},1,1) eq $dr;
  }
}

sub check_is_open($)
{
  my ($f)=@_;
  foreach (keys %open_files)
  {
    error(194,"Open") if uc($open_files{$_}{canonical_name}) eq uc($f);
  }
}

sub save_inf($$$$$$)
{
  my ($path,$fname,$load,$exec,$lock,$crc)=@_;
  my $fh=new FileHandle ">$path";
  printf $fh "%-9s %6X %6X %s CRC=%04X",
                           "$fname",
                           $load,
                           $exec,
                           $lock,
                           $crc;
  close($fh);
}

# We're passed a fill Linux path name; just open the file and slurp
# the contents
sub load_file($)
{
  my ($file)=@_;
  debug(35,"Loading data from $file");

  my $fh=new FileHandle "<$file";
  error(214,"Not found") unless $fh;
  local $/ = undef;
  my $r=<$fh>;
  $fh->close();
  my @d=map { ord($_) } split(//,$r);  # Store data as bytes
  return @d;
}

# Convert a filename into :drive.dir.filename
sub canonify($)
{
  my ($d)=@_;
  debug(30,"To canonify $d");
  my ($ldrive,$ldir,$file)=($drive,$dir,"");
  if ($d=~/^:(.)\.(.*)$/)
  {
    check_drive($1);
    debug(40,"Found drive $1");
    $ldrive=$1;
    $d=$2;
  }
  if ($d=~/^(.)\.(.*)$/)
  {
    debug(40,"Found dir $1");
    $ldir=uc($1);
    $d=uc($2);
  }
  $file=":$ldrive.$ldir.$d";
  debug(30,"Filename is $file");
  return $file;
}

# Convert :drive.dir.file into components
sub decompose($)
{
  my ($f)=@_;
  my ($ldrive,$ldir,$fname)=($f=~/^:([\d|L])\.(.)\.(.*)$/);
  debug(30,"Decompose $f becomes drive=$ldrive dir=$ldir name=$fname");
  check_drive($ldrive);
  return($ldrive,$ldir,$fname);
}

sub get_disks()
{
  my %d=();
  my $dsk=-1;

  # First let's look for any disks beginning with a number; we'll put
  # that in that slot
  foreach (<$disk_base/[0-9]*.*>)
  {
    next unless -d $_;
    s/^.*\///;
    my ($num,$name)=(/^(\d+)\.(.*)$/);
    next unless defined($num);
    $num=int($num);
    $d{$num}=$_;
    debug(30,"$_ means disk $num name $name");
    $dsk=$num if $num>$dsk;
  }
  $dsk++;
  # Now any other disk
  foreach (sort <$disk_base/*>)
  {
    next unless -d $_;
    my $name=$_; $name=~s/^.*\///;
    next if $name =~ /^\d+\./;  # Already handled
    $d{$dsk}=$name;
    debug(30,"$_ means disk $dsk name $name");
    $dsk++;
  }
  return(%d);
}

sub set_opt4($)
{
  my ($y)=@_;
  $y=$y & 3;  # Only 0 1 2 3 allowed
  error(199,"No disk in drive $drive") unless defined($disks{$drive});
  my $fh=new FileHandle "> $disk_base/$disks{$drive}/.opt4";
  print $fh "$y\n";
  close($fh);
}

sub get_cat($;$)
{
  my ($dr,$want_opt)=@_;
  error(199,"No disk in drive $dr") unless defined($disks{$dr});

  my %files;
  # BBC will display files with different cases, but is case insensitive
  foreach (<$disk_base/$disks{$dr}/*>)
  {
    next unless -f $_;
    next if /\.inf$/;
    debug(30,"Found $_");
    my $n=$_; $n=~s!^.*/!!;
    my $k=uc($n); $k="\$.$k" unless $k=~/^.\./;
    debug(35,"Catalog key is $k");
    my $name=$n;
    my $load=0;
    my $exec=0;
    my $locked=1;
    my $size=0;
    my $crc=0;
    my @s=stat($_);
    if (@s)
    {
      $locked=0 if ($s[2] & 0200);
      $size=$s[7];
    }
    if ( -e "${_}.inf")
    {
      my $fh=new FileHandle "< ${_}.inf";
      if (!$fh)
      {
        error(199,"Disk read fault: Can not open ${_}.inf");
      }

      my $line=<$fh>;  chomp($line);
      close($fh);
      my ($fn,$l,$e,$lck,$fcrc)=split(/\s+/,$line);
      error(199,"Bad load address $load for $_") unless $l=~/^[0-9A-Fa-f]+$/;
      error(199,"Bad exec address $exec for $_") unless $e=~/^[0-9A-Fa-f]+$/;
      $k=uc($fn); debug(35,"Key updated to $k");
      $name=$fn;
      $load=$l;
      $exec=$e;
      $locked=1 if $lck =~ /Locked/i;
      $crc=$fcrc;
      $crc=$lck if $lck=~ /CRC/;
      $crc=~s/CRC=//;
      $crc=hex($crc);
    }
    $name='$.' . $name unless $name=~ /^.\./;
    $files{$k}{name}=$name;
    $files{$k}{load}=hex("0x$load");
    $files{$k}{exec}=hex("0x$exec");
    $files{$k}{size}=$size;
    $files{$k}{locked}=$locked;
    $files{$k}{crc}=$crc;
    $files{$k}{fullpath}=$_;
  }
  
  if ($want_opt)
  {
    my $o=0;
    my $fh=new FileHandle "< $disk_base/$disks{$dr}/.opt4";
    if ($fh)
    {
      chomp($o=<$fh>);
      close($fh);
    }
    $files{".opt4"}=$o;
  }
 
  return (%files);
}

# Pass a pattern (eg ":2.*.*") and this will load a catalog from
# the relevant disk and tag each file with "selected" if it matches
# the pattern
sub match_files($)
{
  my ($pattern)=@_;
  my ($ldrive,$ldir,$fname)=decompose(canonify($pattern));
  debug(10,"Searching for $ldrive $ldir $fname");
  error(204,"Bad name - missing file component") unless $fname;

  # Now let's convert the Beeb pattern to a regex
  my $want="$ldir.$fname";
  $want=~ s/(\W)/\\$1/g; # as per perlre
  # * means .*
  # # means .
  # at this point these characters will have a \ in front
  $want=~s/\\#/./g;
  $want=~s/\\\*/.*/g;
  debug(20,"Regex is $want");

  my %files=get_cat($ldrive);
  foreach (keys %files)
  {
    $files{$_}{matched}=($files{$_}{name} =~ /^$want$/i);
    debug(30,"$_ $files{$_}{name} matches $files{$_}{matched}");
  }
  return(%files);
}

sub fname_to_cat($)
{
  my ($f)=@_;
  my %s=%$f;
  my %r;
  foreach (keys %s)
  {
    next if /^\./;
    $r{$s{$_}{name}}=$_;
  }
  return(%r);
}

# ==========================================================================
# SECTION: OSFSC
# ==========================================================================

sub OSFSC()
{
  my $x=read_byte;
  my $y=read_byte;
  my $a=read_byte;
  debug(1,"a=$a x=$x y=$y");

  # What call was requested?
     if ($a == 0) { OSFSC_opt($x,$y); }
  elsif ($a == 1) { OSFSC_checkeof($x,$y); }
  elsif ($a == 2) { OSFSC_run($x,$y); }
  elsif ($a == 3) { OSFSC_cmd($x,$y); }
  elsif ($a == 4) { OSFSC_run($x,$y); }
  elsif ($a == 5) { OSFSC_cat($x,$y); }
  elsif ($a == 7) { OSFSC_handles($x,$y); }
  elsif ($a == 9) { OSFSC_ex(); }
  elsif ($a == 10) { OSFSC_info(); }
  elsif ($a == 11) { OSFSC_librun($x,$y); }
  elsif ($a == 255) { OSFSC_boot($x,$y); }
  else {
    # Anything else we ignore
    debug(1,"Ignoring this call");
    send_bytes(255,$y,$x);
  }
}


sub OSFSC_opt($$)
{
  my ($x,$y)=@_;
  debug(1,"*OPT $x,$y");
  $opt{$x}=$y;
  if ($x == 4) { set_opt4($y); }
  if ($x == 5) { $DEBUG=$y; debug(1,"Debug level set to $y"); }
  if ($x == 6) { $PAUSE_AFTER=int($y/16); $PAUSE_DELAY=$y%16; $PAUSE_DELAY=2 unless $PAUSE_DELAY; debug(1,"Pause set to $PAUSE_AFTER $PAUSE_DELAY"); }
  send_bytes(255,$y,$x);
}

sub OSFSC_checkeof($$)
{
  my ($x,$y)=@_;

  my $h=0;
  if (defined($open_files{$x}))
  {
    $h=255 if $open_files{$x}{ptr} >= $open_files{$x}{ext};
  }
  send_bytes(255,$y,$h);
}

sub OSFSC_cmd($$)
{
  # Tell client to send us the command
  send_byte(127);
  my $string=read_string;
  debug(1,"*Command: $string");

  # Now let's split the command
  my ($cmd,@args)=split(/\s+/,$string);
  # If there's a . in cmd then split on it and munge the array
  # *DRI.2 is really DRI. 2
  # (this is safe 'cos we don't have any . in our internal commands,
  # and so things like :2.$.foo won't match and $.foo _should_ attempt
  # to match command "$." with an argument of foo ("i.2" for example)
  if ($cmd=~/^([^\.]*\.)(.*)$/)
  {
    $cmd=$1;
    @args=($2,@args);
  }
  debug(30,"Checking for $cmd");
  $cmd=uc($cmd); # Case independent

  if (cmd_match($cmd,"HSTATUS"))
  {
    debug(1,"Handling *HSTATUS");
    my $what=$args[0] || 'DF';
    my $r="";
    if ($what =~ /D/i)
    {
      foreach (0..9)
      {
        $r .= "Disk $_: " . ($disks{$_} || "") . "\r\n";
      }
      $r .= "\r\nCurrent drive: $drive\r\n" .
            "Current directory: $dir (:$drive.$dir)\r\n" .
            "Current library: $lib\r\n" .
            "\r\n";
    }
    if ($what =~ /F/i)
    {
      $r .= "Open files:\r\n### Filename     PTR#   EXT#   How D\r\n";
      foreach (sort keys %open_files)
      {
        my $d=$open_files{$_}{dirty}?"Y":"N";
        my $m=$open_files{$_}{mode};
        my $m2="IN "; $m2="OUT" if $m == 0x80; $m2="UP "if $m == 0xc0;
        my $f=$open_files{$_}{fullpath}; $f=~s/^$disk_base.//;
        $f .= "\r\n" . (" " x 16) if length($f)>12;
      
        $r .= sprintf("%3s %-12s %06X %06x %-3s %s\r\n",
                       $_,$f,
                       $open_files{$_}{ptr},
                       $open_files{$_}{ext},
                       $m2,$d);
      }
      $r .= "\r\n";
    }

    send_output("$r\r\n");
##    send_byte(0);
  }
  elsif (cmd_match($cmd,"DCAT"))
  {
    debug(1,"*DCAT");
    my %d=get_disks;
    my $res="Disks available:\r\n";

    foreach (sort { $a <=> $b} keys %d)
    {
      $res .= sprintf("%6d: %s\r\n",$_,$d{$_});
    }
    send_output($res);
  }
  elsif (cmd_match($cmd,"DCREATE"))
  {
    debug(1,"Handling *DCREATE");
    my $want=need_arg($args[0]);
    if ($want =~ /^(\d+)\./)
    {
      my $dnum=$1;
      my %d=get_disks;
      error(199,"Already have a disk in slot $dnum") if $d{$dnum};
    }
    my $want2=$want;
    $want2=~tr/ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789_+.!@$%,-/_/c;
    error(199,"Bad characters present") unless $want eq $want2;
    mkdir("$disk_base/$want");
    debug(10,"Made $disk_base/$want");
    send_output("$want created\r\n");
##    send_byte(0);
  }    
  elsif (cmd_match($cmd,"DIN"))
  {
    debug(1,"Handling *DIN");
    find_and_load_disk(@args);

    send_byte(0);
  }
  elsif (cmd_match($cmd,'DOUT'))
  {
    my $dr=check_drive($args[0]);
    check_drive_in_use($dr);
    undef($disks{$dr});
    send_byte(0);
  }
  elsif (cmd_match($cmd,'DBOOT'))
  {
    find_and_load_disk(0,@args);
    # We now need to send a dummy program that does
    # *fx 255 7, followed by JMP (&FFFC)
    my @code=(0xA9,0xFF,0xA2,0x07,0xA0,000,0x20,0xF4,0xFF,0x6C,0xFC,0xFF);

    send_data_to_memory(0x0900,@code);
    send_cmd(0xc0);    # set execution address
    send_addr(0x0900);
    send_byte(0x80);   # do it!
  } 
  elsif (cmd_match($cmd,'DRIVE'))
  {
    my $dr=check_drive($args[0]);
    $drive=$dr;
    send_byte(0);
  }
  elsif (cmd_match($cmd,'DIR'))
  {
    my $d=need_arg($args[0]);

    my ($dr,$di)=decompose(canonify("$d.dummy"));
    error(206,"Bad dir") unless $di;
    ($drive,$dir)=($dr,$di);

    send_byte(0);
  }
  elsif (cmd_match($cmd,'LIB'))
  {
    my $d=need_arg($args[0]);

    my ($ldrive,$ldir)=decompose(canonify("$d.dummy"));
    $lib=":$ldrive.$ldir";
    send_byte(0);
  }
  elsif (cmd_match($cmd,'HRESET'))
  {
    debug(1,"Restarting!");
    send_output("Host is restarting!\r\n");
    sleep(1) if $UPURS;  # Ensure data is sent!
    exec($0,@ORIG_ARGV);
  }
  elsif (cmd_match($cmd,'VERSION'))
  {
    send_output("TubeHost version: $VERSION\r\nYour port is: $socket_path\r\nHost: " . `hostname` . "\rYou are: $identity\r\n");
  }
  elsif (cmd_match($cmd,'INFO'))
  {
    OSFSC_do_info($args[0]);
  }
  elsif (cmd_match($cmd,'IAM'))
  {
    my $x=join(" ",@args);
    
    $identity=$x || 'Anonymous';
    send_output("You are: $identity\r\n");
  }
  elsif (cmd_match($cmd,'DELETE'))
  {
    my $string=extract_filename(need_arg($args[0]));
    my $canon=canonify($string);
    check_is_open($canon);
    my ($ldrive,$ldir,$lfile)=decompose($canon);
    my $want=uc($ldir) . "." . uc($lfile);
    my %files=get_cat($ldrive);
    error(214,"Not found") unless defined($files{$want});
    error(195,"Locked") if $files{$want}{locked};
    unlink($files{$want}{fullpath});
    unlink($files{$want}{fullpath} . ".inf");
    send_byte(0);
  }
  elsif (cmd_match($cmd,'RENAME'))
  {
    my $string=extractfilename(need_arg($args[0]));
    my $canon=canonify($string);
    check_is_open($canon);
    my ($ldrive,$ldir,$lfile)=decompose($canon);
    my $want=uc($ldir) . "." . uc($lfile);

    my $target=extract_filename(need_arg($args[1]));
    my ($tdrive,$tdir,$tfile)=decompose(canonify($target));
    my $twant=uc($tdir) . "." . uc($tfile);

    error(205,"Bad drive") unless $ldrive == $tdrive;

    debug(10,"Renaming $want to $target\n");
        
    # Silently ignore renames that do nothing, so only do
    # stuff if the keys are different
    if ($want ne $twant)
    {

      my %files=get_cat($ldrive);
      error(214,"Not found") unless defined($files{$want});
      error(195,"Locked") if $files{$want}{locked};
      error(196,"Exists") if defined($files{$twant});

      # Seems sane, lets do it.
      my $newpath=gen_path($ldrive,"$tdir.$tfile");
      rename($files{$want}{fullpath},$newpath);
      unlink($files{$want}{fullpath} . ".inf");
      save_inf("$newpath.inf","$tdir.$tfile",
                   $files{$want}{load},
                   $files{$want}{exec},
                   $files{$want}{locked},
                   $files{$want}{crc});
    }
    send_byte(0);
  }
  elsif (cmd_match($cmd,'ACCESS'))
  {
    my $string=extract_filename(need_arg($args[0]));
    my $lock=uc($args[1] || '');
    error(207,"Bad attribute") unless $lock eq 'L' || $lock eq '';
    my $canon=canonify($string);
    check_is_open($canon);
    my ($ldrive,$ldir,$lfile)=decompose($canon);
    my $want=uc($ldir) . "." . uc($lfile);
    my %files=get_cat($ldrive);
    error(214,"Not found") unless defined($files{$want});
    $files{$want}{locked}=$lock?"Locked":"";
    save_inf($files{$want}{fullpath} . ".inf",
             $files{$want}{name},
             $files{$want}{load},
             $files{$want}{exec},
             $lock?"Locked":"",
             $files{$want}{crc});
    send_byte(0);
  }
  elsif (cmd_match($cmd,'DMP'))
  {
    my $string=extract_filename(need_arg($args[0]));
    my $canon=canonify($string);
    check_is_open($canon);
    my ($ldrive,$ldir,$lfile)=decompose($canon);
    my $want=uc($ldir) . "." . uc($lfile);
    my %files=get_cat($ldrive);
    error(214,"Not found") unless defined($files{$want});
    my @d=load_file($files{$want}{fullpath});

    my $r="";
    my $offset=0;

    my $cnt=0;
    my ($hex,$ascii);
    while (@d)
    {
      my $ch=shift @d;
      my ($nhex,$nch);
      $nhex=to_hex($ch);
      if ($ch < 32 || $ch > 126) { $nch="."; } else { $nch=chr($ch); }
      $hex .= "$nhex ";
      $ascii .= $nch;
      $cnt++;
      if ($cnt == 8)
      {
        $r .= sprintf "%04X %s %s\r\n",$offset,$hex,$ascii;
        $hex="";
        $ascii="";
        $offset += 8;
        $cnt=0;
      }
    }
    if ($cnt)
    {
      $hex .= "   " x (8-$cnt);
      $r .= sprintf "%04X %s %s\r\n",$offset,$hex,$ascii;
    }
    send_output($r);
  }
  elsif (cmd_match($cmd,"FOO"))
  {
    my $d=int(need_arg($args[0]));
    my $r="X" x $d;
    send_output($r);
  }
  else
  {
    # Oh well.. we don't know this command; treat as *RUN
    debug(1,"Not handled by us; passing to *RUN routine");
    OSFSC_do_run($string);
  }
}

sub find_and_load_disk($)
{
  my ($id,$name)=@_;

  my $dr=check_drive($id);
  check_drive_in_use($dr);
  my $dn=need_arg($name);

  debug(10,"Looking for $dn");
  my %d=get_disks;
  if ($dn =~ /^\d+$/)
  {
    check_disk_num($dn,\%d);
    $disks{$dr}=$d{$dn};
  }
  else
  {
    # Let's find the disk with this name
    my $found=0;
    my $want=uc($dn);
    # Strip off quotes
    $want=~s/^"(.*)"$/$1/;

    foreach (sort keys %d)
    {
      my $name=uc($d{$_});
      # strip off potential disk number
      $name =~ s/^[0-9]+\.//;
      if ( (uc($d{$_}) eq $want) || ($name eq $want) )
      {
        $disks{$dr}=$d{$_};
        $found=1;
        last;
      }
    }
    error(214,"File or path not found") unless $found;
  }
  debug(1,"Drive $dr set to $disks{$dr}");
}

sub OSFSC_handles($$)
{
  debug(1,"Request for filehandles");
  send_bytes(255,$FH_LOW,$FH_HIGH);
}

sub OSFSC_run($$)
{
  # Tell client to send us the command
  send_byte(127);
  my $string=read_string;
  debug(1,"File to run: $string");
  OSFSC_do_run($string);
}

sub OSFSC_librun($$)
{
  # Tell client to send us the command
  send_byte(127);
  my $string=read_string;
  debug(1,"File to lib run: $string");
  OSFSC_do_run($string,1);
}

sub OSFSC_do_run($;$)
{
  my ($string,$lib_only)=@_; $lib_only+=0;
  $string=extract_filename($string);
  debug(1,"Run from filesystem $string - $lib_only");
  my ($ldrive,$ldir,$lfile,$want,%files);

  if (!$lib_only)
  {
    ($ldrive,$ldir,$lfile)=decompose(canonify($string));
    $want=uc($ldir) . "." . uc($lfile);
    %files=get_cat($ldrive);
  }

  if (!defined($files{$want}) || $lib_only)
  {
    # Let's check the library
    debug(20,"Can not find $want on $ldrive, checking library");
    ($ldrive,$ldir,$lfile)=decompose(canonify("$lib.$string"));
    $want=uc($ldir) . "." . uc($lfile);
    debug(20,"... want $want from $ldrive instead");
    %files=get_cat($ldrive);
  }

  # If we don't find it here, let's always look in _Library
  if (!defined($files{$want}))
  {
    debug(20,"Can not find $want in library, checking _Library");
    ($ldrive,$ldir,$lfile)=decompose(canonify(":L.$.$string"));
    $want=uc($ldir) . "." . uc($lfile);
    debug(20,"... want $want from $ldrive instead");
    %files=get_cat($ldrive);
  }

  error(254,"Bad command") unless defined($files{$want});

  send_file($files{$want}{fullpath},$files{$want}{load});
  send_cmd(0xc0);    # set execution address
  send_addr($files{$want}{exec});
  send_byte(0x80);   # do it!
  return;
}

sub OSFSC_info()
{
  # Tell client to send us the data
  send_byte(127);
  my $string=read_string;
  OSFSC_do_info($string);
}

sub OSFSC_ex()
{
  # Tell client to send us the data
  send_byte(127);
  my $string=extract_filename(read_string);
  my ($ldr,$ldir)=decompose(canonify("$string.dummy"));
#
#  # This should be a directory or a file/directory so let's just add stuff
#  # and then canonify
#  $string .= "." unless $string =~ /\.$/;
#  $string .= "$dir.*";
#  my ($ldr,$ldir,$f)=decompose(canonify($string));
  OSFSC_do_info(":$ldr.$ldir.*");
}

sub OSFSC_do_info($)
{
  my ($d)=@_;

  debug(10,"*INFO $d");
  my %f=match_files(need_arg($d));
  my $r;
  foreach (sort { $f{$a}{name} <=> $f{b}{name} } keys %f)
  {
    next unless $f{$_}{matched};
    $r .= sprintf "%-10s  %s  %06X %06X %06X\r\n",
                        $f{$_}{name},
                        ($f{$_}{locked}?"L":" "),
                        $f{$_}{load},
                        $f{$_}{exec},
                        $f{$_}{size};
  }
  error(214,"Not found") unless $r;
  send_output($r);
}

sub format_filename($$$)
{
  my ($name,$index,$files)=@_;
  my %f=%$files;
  my $l=$f{$index}{locked};
  $name=~s/^../  / if uc(substr($name,0,2)) eq "$dir.";
  if ($l)
  {
    return(sprintf("%-10s L",$name));
  }
  return($name);
}

sub opt_str($)
{
  my ($o)=@_;
  $o=0 unless $o;
     if ($o == 1) { return "1 (LOAD)"; }
  elsif ($o == 2) { return "2 (RUN)"; }
  elsif ($o == 3) { return "3 (EXEC)"; }
  else { return "0 (off)";}
}

sub OSFSC_cat($)
{
  my ($short)=@_;

  # Tell client to send us the data
  send_byte(127);
  my $string=read_string;
  my $dr=$drive;
  if ($string)
  {
    $dr=check_drive($string);
  }
  my %files=get_cat($dr,1);
  my %f=fname_to_cat(\%files);
  my $r="Drive $dr ($disks{$dr})\r\n" .
        "Dir. :$drive.$dir   Lib. $lib  Opt: " . opt_str($files{".opt4"}) . "\r\n";
  my $c=0;
  my $l=0;
  foreach (sort keys %f)
  {
    next unless uc(substr($_,0,2)) eq "$dir.";

    $r .= $c?" " x (18-$l):"\r\n";
    $_=format_filename($_,$f{$_},\%files);
    $r .= "  $_";
    $l=length($_);
    $c=1-$c;
    $c=0 if $l>18;
  }
  $r .= "\r\n";
  $c=0;
  foreach (sort keys %f)
  {
    next if uc(substr($_,0,2)) eq "$dir.";

    $r .= $c?" " x (18-$l):"\r\n";
    $_=format_filename($_,$f{$_},\%files);
    $r .= "  $_";
    $l=length($_);
    $c=1-$c;
    $c=0 if $l>18;
  }
  $r .= "\r\n";
  send_output($r);
}

sub OSFSC_boot($$)
{
  my ($x,$y)=@_;
  debug(1,"ShCtrl-BREAK");

  # User pressed shift-break or control-break.
  # We need to close all open files
  # Set drive,dir,lib
  foreach (keys %open_files) { OSFIND_close($_); }
  $drive=0;
  $dir='$';
  $lib=':0.$';
  my %f=get_cat($drive,1);
  $y=$f{".opt4"} || 0;
  send_bytes(255,$y,$x);
}

# ==========================================================================
# SECTION: OSFILE
# ==========================================================================

# This function doesn't nicely decompose into smaller onces 'cos there's
# a lot of shared variables that'd need passing around.  So it's just a
# large if/then/else structure.  Oh well...

sub OSFILE()
{
  debug(1,"OSFILE");
  # OSFILE data is sent backwards
  my $end=read_addr;  # aka attr
  my $save=read_addr; # aka len
  my $exec=read_addr;
  my $load=read_addr;
  my $file=extract_filename(read_string);
  my $cmd=read_byte;

  my $canon=canonify($file);
  check_is_open($canon);
  my ($ldrive,$ldir,$lfile)=decompose($canon);

  my $key=uc("$ldir.$lfile");  debug(35,"OSFILE key $key");
  my %cat=get_cat($ldrive);

  # If we're not saving a new file then it must exist
  error(214,"File not found") if $cmd != 0 && $cmd != 7 && !$cat{$key};

  # What does the existing file hold?
  my ($fname,$fload,$fexec,$fsave,$fend,$fullpath,$flocked,$crc);

  # Even for save we might care about the locked file (overwrite)
  # so we'll always get the attributes
  if ($cat{$key})
  {
    $fullpath=$cat{$key}{fullpath};
    $fname=$cat{$key}{name};
    $fload=$cat{$key}{load};
    $fexec=$cat{$key}{exec};
    $fsave=$cat{$key}{size};
    $flocked=$cat{$key}{locked};
    $crc=$cat{$key}{crc};
    $fend=$flocked?$FILE_LOCKED:$FILE_UNLOCKED;
  }

  # Anything except "load" and "read file info" rewrites, so...
  error(195,"Locked") if $flocked && $cmd != 255 && $cmd != 5;

  if ( $cmd == 255)  # LOAD
  {
    # Get the contents.
    my @data=load_file($fullpath);

    # If the user didn't specify a load address, use the one in the file
    $load=$fload unless $load;
    send_data_to_memory($load,@data);
  }
  elsif ($cmd == 0 || $cmd == 7) # SAVE
  {
    # What file should we save to?
    # If this is a new file, generate the name
    $fullpath=gen_path($ldrive,"$ldir.$lfile") unless $fullpath;
    
    # Let's grab the data
    debug(1,"SAVE :$ldrive.$ldir.$lfile to $fullpath");
    my $len=$end-$save;
    debug(20,"We want $len characters");
    # For command 0 we ask the client to send data, else we just create
    # zero buffer
    my $data="";
    if (!$cmd)
    {
      $data=read_data_from_memory($save,$len);
    }
    else
    {
      $data .= chr(0) x $save;
    }
    my $fh=new FileHandle ">$fullpath";
    error(201,"Can not open file") unless $fh;
    print $fh $data;
    close($fh);
    $crc=CalcCRC($data);
    save_inf("$fullpath.inf","$ldir.$lfile",$load,$exec,"",$crc);
    $end=$FILE_UNLOCKED;
    $save=$len;
  }
  elsif ($cmd == 1) # Save parameter block to file
  {
    save_inf("$fullpath.inf","$ldir.$lfile",$load,$exec,($end & 5)?$1:0,$crc);
  }
  elsif ($cmd == 2) # Change load address
  {
    save_inf("$fullpath.inf",$fname,$load,$fexec,$flocked,$crc);
    ($exec,$end)=($fexec,$flocked);
  }
  elsif ($cmd == 3) # Change exec address
  {
    save_inf("$fullpath.inf",$fname,$fload,$exec,$flocked,$crc);
    ($load,$end)=($fload,$flocked);
  }
  elsif ($cmd == 4) # Change load/exec address
  {
    save_inf("$fullpath.inf",$fname,$load,$exec,$flocked,$crc);
    $end=$flocked;
  }
  elsif ($cmd == 5)
  {
    ($load,$exec,$end,$save)=($fload,$fexec,$flocked,$fsave);
  }
  elsif ($cmd == 6)
  {
    unlink($fullpath);
    unlink("$fullpath.inf");
  }
  else
  {
    error(214,"You want $cmd for $file");
  }

  # Send results
  send_byte(1);    # File found; directories aren't possible
  send_addr($end);
  send_addr($save);
  send_addr($exec);
  send_addr($load);
}

# ==========================================================================
# SECTION: OSFIND
# ==========================================================================
sub OSFIND()
{
  my $fn=read_byte;

  # function 0 (close) takes a byte; everything else takes a file
  my $parm=$fn?extract_filename(read_string):read_char;

  debug(1,"OSFIND $fn $parm");

  if ($fn !=0 && $fn != 0x40 && $fn != 0x80 && $fn != 0xc0)
  {
    debug(10,"Bad command $fn; returning 0");
    send_byte(0);
    return;
  }

  if ($fn == 0)
  {
    debug(10,"CLOSE #$parm");
    my $r="";
    if ($parm == 0)
    {
      foreach (keys %open_files) { $r .= OSFIND_close($_); }
    }
    else
    {
       $r=OSFIND_close($parm);
    }
    error(201,$r) if $r;
    send_byte(0x7f);
    return;
  }
  
  my $canon=canonify($parm);
  check_is_open($canon);
  my ($ldrive,$ldir,$lfile)=decompose($canon);
  my $key=uc("$ldir.$lfile");  debug(35,"key $key");
  my %cat=get_cat($ldrive);
  debug(10,"File wanted $parm == $canon");

  # OPENIN or OPENUP file must exist; ifit doesn't, return 0
  if (($fn == 0x40 || $fn == 0xc0) && !$cat{$key})
  {
    send_byte(0);
    return;
  }
  #### error(214,"File not found") if $fn == 0x40 && !$cat{$key};

  # OPENOUT/UP file either doesn't exist or is not locked
  error(195,"Locked") if $fn > 0x40 && $cat{$key}{locked};

  # what file handle to use.  We can be sure this loop will terminate 'cos
  # we'll never open $open_files{$FH_HIGH+1}
  my $open=$FH_LOW;
  while (defined($open_files{$open})) { $open++; }
  error(192,"Too may open files") if $open > $FH_HIGH;

  my $fullpath=$cat{$key}{fullpath};
  # If the file doesn't exist, create it (OPENIN must already exist)
  if (!$fullpath)
  {
    $fullpath=gen_path($ldrive,"$ldir.$lfile");
    debug(10,"Creating new file $fullpath");
    my $fh=new FileHandle ">$fullpath";
    error(201,"Can not open file") unless $fh;
    close($fh);
    my $crc=CalcCRC("");
    save_inf("$fullpath.inf","$ldir.$lfile",0,0,"",$crc);

    # We know the data!
    @{ $open_files{$open}{data} }=();
    $open_files{$open}{ext}=0;
    $open_files{$open}{dirty}=1;
  }
  elsif ($fn != 0x80)
  {
    # File exists; use it
    my @data=load_file($cat{$key}{fullpath});
    @{ $open_files{$open}{data} }=@data;
    $open_files{$open}{ext}=@data;
    $open_files{$open}{dirty}=0;
  }
  else
  {
    # OPENOUT; always truncate
    @{ $open_files{$open}{data} }=();
    $open_files{$open}{ext}=0;
    $open_files{$open}{dirty}=1;
  }
  
  # OK!
  $open_files{$open}{canonical_name}=$canon;
  $open_files{$open}{fullpath}=$fullpath;
  $open_files{$open}{ptr}=0;
  $open_files{$open}{mode}=$fn;

  send_byte($open);
}

sub OSFIND_close($)
{
  my ($h)=@_;
  debug(10,"Close handle $h");
  return unless defined($open_files{$h});

  my $err=OSARGS_sync_file($h);
  return($err) if $err;

  delete($open_files{$h});
  return("");
}
 
# ==========================================================================
# SECTION: BPUT
# ==========================================================================
sub OSBPUT()
{
  my $h=read_byte;
  my $d=read_byte;
  debug(1,"BPUT #$h,$d");
  error(222,"Channel") unless defined($open_files{$h});
  error(193,"Read only") if $open_files{$h}{mode} == 0x40;
  my $ptr=$open_files{$h}{ptr};
  my $ext=$open_files{$h}{ext};
  my @data=@{ $open_files{$h}{data} };
  $data[$ptr++]=$d;
  $ext=($ptr>$ext)?$ptr:$ext;
  @{ $open_files{$h}{data} }=@data;
  $open_files{$h}{ptr}=$ptr;
  $open_files{$h}{ext}=$ext;
  $open_files{$h}{dirty}=1;
  send_byte(0x7f);
}

# ==========================================================================
# SECTION: BGET
# ==========================================================================
sub OSBGET()
{
  my $h=read_byte;
  debug(1,"BGET #$h");
  error(222,"Channel") unless defined($open_files{$h});
  my $ptr=$open_files{$h}{ptr};
  my $ext=$open_files{$h}{ext};
  my @data=@{ $open_files{$h}{data} };
  if ($ptr >= $ext)
  {
    debug(10,"EOF #$h");
    send_bytes(0x80,0);
  }
  else
  {
    send_bytes(0,$data[$ptr++]);
    $open_files{$h}{ptr}=$ptr;
  }
}

# ==========================================================================
# SECTION: OSARGS
# ==========================================================================

sub OSARGS
{
  my $handle=read_byte;
  my $param=read_addr;
  my $fn=read_byte;
  debug(1,"OSARGS $fn $handle $param");

  if ($handle == 0)
  {
    if ($fn == 0) { $fn=9; }
    elsif ($fn == 255) { foreach (keys %open_files) { OSARGS_sync_file($_); } }
    else { debug(10,"Ignoring unknown OSARGS y=0"); }
  }
  else
  {
    error(222,"Channel") unless defined($open_files{$handle});
       if ($fn == 0) { $param=$open_files{$handle}{ptr}; }
    elsif ($fn == 1) { OSARGS_set_ptr($handle,$param); }
    elsif ($fn == 2) { $param=$open_files{$handle}{ext}; }
    elsif ($fn == 3) { OSARGS_set_ext($handle,$param); }
    elsif ($fn == 255) { OSARGS_sync_file($handle); }
    else { debug(10,"Ignoring unknown OSARGS y=$handle"); }
  }
  send_byte($fn); send_addr($param);
}
  
sub OSARGS_sync_file($)
{
  my ($h)=@_;

  debug(10,"Sync file $h");
  # If this file is dirty, write it out
  if ($open_files{$h}{dirty})
  {
    my $f=$open_files{$h}{fullpath};
    my @data=@{$open_files{$h}{data}};
    my $fh=new FileHandle ">$f";
    if (!$fh)
    {
      debug(5,"Closing $f failed\n");
      delete($open_files{$h});
      return "Data not flushed to $f\r\n";
    }
    else
    {
      debug(10,"Flushing $f");
      my $data=join("",map { chr($_) } @data);
      print $fh $data;
      close($fh);
      my $crc=CalcCRC($data);
      my $lname=$open_files{$h}{canonical_name};
      $lname=~s/:\d\.//;
      save_inf("$f.inf","$lname",0,0,"",$crc);
    }
  }
  $open_files{$h}{dirty}=0;
  return "";
}

sub OSARGS_set_ptr($$)
{
  my ($handle,$param)=@_;

  debug(10,"PTR#$handle=$param");
  my $ext=$open_files{$handle}{ext};
  OSARGS_set_ext($handle,$param) if $param>$ext;
  $open_files{$handle}{ptr}=$param;
}

sub OSARGS_set_ext($$)
{
  my ($handle,$param)=@_;

  debug(10,"EXT#$handle=$param");
  error(193,"Read only") if $open_files{$handle}{mode} == 0x40;
  my ($ext)=$open_files{$handle}{ext};

  if ($param>$ext)
  {
    # Make larger
    foreach ($ext..$param-1)
    {
      push(@{ $open_files{$handle}{data} },0);
    }
    $open_files{$handle}{dirty}=1;
  }
  elsif ($param < $ext)
  {
    # Shrink the file
    $open_files{$handle}{dirty}=1;
    @{ $open_files{$handle}{data} }=(@{ $open_files{$handle}{data} })[0..$param-1];
  }
  $open_files{$handle}{ext}=$param;
}

# ==========================================================================
# SECTION: OSGBPB
# ==========================================================================

sub OSGBPB()
{
  # Data passed to this call
  my $ptr=read_addr;
  my $size=read_addr;
  my $load=read_addr;
  my $fh=read_byte;
  my $fn=read_byte;
  my $carry=0;

  debug(1,"OSFIND $fn handle=$fh load=$load size=$size ptr=$ptr");
  # If this is a file based call, check the handle is open
  if ($fn < 5)
  {
    error(222,"Channel") if !$open_files{$fh};
    $ptr=$open_files{$fh}{ptr} if $fn == 2 || $fn == 4;
    my $ext=$open_files{$fh}{ext};
    my @data=@{ $open_files{$fh}{data} };
    # Send data to file
    if ($fn == 1 || $fn == 2)
    {
      my $tosave=read_data_from_memory($load,$size);
      while ($ext < $ptr)
      {
        push(@data,0);
        $ext++;
      }

      foreach (split(//,$tosave))
      {
        $data[$ptr++]=ord($_);
      }
      $open_files{$fh}{ptr}=$ptr;
      @{ $open_files{$fh}{data} }=@data;
      $open_files{$fh}{ext}=@data;
      $open_files{$fh}{dirty}=1;

      $load+=$size;
      $size=0;
    }
    else
    {
      my $end=$ptr+$size;
      if ($end>$ext)
      {
        $carry=1;
        $end=$ext;
      }
      my @d=@data[$ptr..$end-1];

      # Eat my data
      send_data_to_memory($load,@d);
      $load += @d;
      $ptr += @d;
      $size -= @d;
    }
  }
  elsif ($fn == 5)
  {
    # This is a kludge to try and make it look DFS-like
    # I don't see DFS actually caring about data sizes.  Now disk titles
    # in this filesystem could be almost unlimited.  I'm gonna force it
    # to be 12 characters long, regardless.
    check_drive($drive);
    my $r=$disks{$drive};
    $r .= " " x 12;
    $r = substr($r,0,12);
    my @data=map { ord($_) } split(//,$r);
    @data=(12,@data,0,$drive);
    send_data_to_memory($load,@data);
  }
  elsif ($fn == 6)
  {
    send_data_to_memory($load,1,ord($drive),1,ord($dir));
  }
  elsif ($fn == 7)
  {
    my ($ldr,$ldir,$f)=decompose(canonify("$lib.dummy"));
    send_data_to_memory($load,1,ord($ldr),1,ord($ldir));
  }
  elsif ($fn == 8)
  {
    # List of files
    debug(1,"Want directory from $ptr for $size entries");
    my %cat=match_files(":$drive.$dir.*");
    my $r="";
    my @names=();
    foreach (sort keys %cat)
    {
      push(@names,$cat{$_}{name}) if $cat{$_}{name};
    }
    if ($ptr > @names)
    {
      $carry=1;
      debug(20,"We only have " . @names . " entries; nothing to return");
    }
    else
    {
      @names=@names[$ptr..@names-1];
      my $want=$size;
      if ($want>@names)
      {
        $want=@names;
        debug(20,"Reducing size request to $want entries");
        $carry=1;
      }
      @names=@names[0..$want-1];
      $size -= @names;
      $ptr += @names;
      foreach my $n (@names)
      {
        $n=~s/^..//;
        $r .= chr(length($n)) . $n;
      }
      my @d=map { ord($_) } split(//,$r);  # Store data as bytes
      send_data_to_memory($load,@d);
      $load += @d;
    }
  }
     
  send_addr($ptr);
  send_addr($size);
  send_addr($load);
  send_byte($fh);
  
  send_byte($carry?0x80:0);
  send_byte($fn);
}


# ==========================================================================
# SECTION: MAIN
# ==========================================================================

# Let's set up what we want
if ($SERIAL)
{
  $socket=new Device::SerialPort($socket_path);
  $socket->baudrate($BAUD);
  $socket->parity('none');
  $socket->databits(8);
  $socket->stopbits(1); 
  $socket->handshake("rts"); 
  $socket->write_settings();
}
elsif ($UPURS)
{
  # I'm not actually convinced Device::SerialPort works properly.
  # For UPURS we're gonna do raw I/O and just cheat.  These
  # are the stty settings we need
  system("stty $BAUD crtscts ignbrk ignpar -icrnl -ixon -opost -isig -icanon -iexten -echo < $socket_path");

  $socket=new FileHandle "+<$socket_path" or die "$socket_path: $!\n";
}
else
{
  $socket = IO::Socket::UNIX->new(
   Type => SOCK_STREAM,
   Peer => $socket_path,
  )
     or die("Can't connect to server $socket_path: $!\n");
}

debug(0,"Connected");

# Initialise disks
my %d=get_disks;
foreach (0..9)
{
  $disks{$_}=$d{$_} if defined($d{$_});
}

MAIN:
while(1)
{
  debug(10,"Character received from client: " . read_char(1));
}
