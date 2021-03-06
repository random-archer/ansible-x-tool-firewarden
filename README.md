# Firewarden

Firewarden is a bash script used to open a program within a private
[Firejail][1] sandbox.

Firewarden will launch the given program within a Firejail sandbox with a
private home directory on a temporary filesystem. Networking is enabled by
default, but may be disabled.

This may be useful for a number of applications, but was created specifically
with Chromium in mind. To open a [shady site][2] within an isolated and
temporary sandbox -- or simply to help further protect your online banking --
prepend your normal command with `firewarden`:

    $ firewarden chromium http://www.forbes.com

When using Firewarden to run `chromium` or `google-chrome`, the script will
prevent the first run greeting, disable the default browser check, and prevent
the [WebRTC IP leak][3].

## Local Files

If the final argument appears to be a local file, Firewarden will copy the file
into a temporary directory. This directory will be used by Firejail as the user
home. After the program has closed, the temporary directory and all of its
contents are deleted.

By default, networking is disabled and a private `/dev` is used when viewing a
local file.

This is particularly useful for mitigating harm caused by opening potentially
malicious files, such as PDF and JPGs. Add it to your mailcap to protect your
system against shady email attachments.

For example, you may want to view the file `~/notatrap.pdf` with the PDF reader `zathura`.

    $ firewarden zathura ~/notatrap.pdf

This is the equivalent of doing:

    $ export now=`date --iso-8601=s`
    $ mkdir -p /tmp/$USER/firewarden/$now
    $ cp ~/notatrap.pdf /tmp/$USER/firewarden/$now/
    $ firejail --net=none --private-dev --private=/tmp/$USER/firewarden/$now zathura notatrap.pdf
    $ rm -r /tmp/$USER/firewarden/$now

## Options

### Network

Networking is enabled by default, unless viewing local files (which most of the
time do not need network access).

The user may explicitly enable or disable network access, overriding the default behavior.

    # deny network access, regardless of the defaults.
    $ firewarden -n ...
    # enable network access, regardless of the defaults.
    $ firewarden -N ...

Optionally, the sandbox may be launched with an isolated network namespace and
a restrictive netfilter. Unless otherwise specified, [NetworkManager][4] will
be used to determine the first connected network interface. This interface will
be used to create the new network namespace.

    # isolate the network, using the first connected interface.
    $ firewarden -i ...
    # isolate the network, using the specified interface.
    $ firewarden -I eth0 ...

When isolating the network, Firejail's default client network filter will be
used in the new network namespace.

```
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
# allow ping
-A INPUT -p icmp --icmp-type destination-unreachable -j ACCEPT
-A INPUT -p icmp --icmp-type time-exceeded -j ACCEPT
-A INPUT -p icmp --icmp-type echo-request -j ACCEPT
# drop STUN (WebRTC) requests
-A OUTPUT -p udp --dport 3478 -j DROP
-A OUTPUT -p udp --dport 3479 -j DROP
-A OUTPUT -p tcp --dport 3478 -j DROP
-A OUTPUT -p tcp --dport 3479 -j DROP
COMMIT

```


### /dev

Optionally, a new `/dev` can be created to further restrict the sandbox. This
has the effect of preventing access to audio input and output, as well as any
webcams. It is enabled by default when viewing local files.

    # create a private /dev, regardless of the defaults.
    $ firewarden -d ...
    # do not create a private /dev, regardless of the defaults.
    $ firewarden -D ...

## Examples

    $ firewarden -d -i chromium https://www.nsa.gov/ia/ &
    $ firewarden zathura /mnt/usb/nsa-ant.pdf &
    $ firewarden chromium https://www.youtube.com/watch?v=bDJb8WOJYdA &
    $ firejail --list
    630:pigmonkey:/usr/bin/firejail --private --net=enp0s25 --netfilter --private-dev chromium --no-first-run --no-default-browser-check https://www.nsa.gov/ia/
    31788:pigmonkey:/usr/bin/firejail --private=/tmp/pigmonkey/firewarden/2016-01-31T16:09:14-0800 --net=none --private-dev zathura nsa-ant.pdf
    32255:pigmonkey:/usr/bin/firejail --private chromium --no-first-run --no-default-browser-check https://www.youtube.com/watch?v=bDJb8WOJYdA


[1]: https://github.com/netblue30/firejail
[2]: http://www.engadget.com/2016/01/08/you-say-advertising-i-say-block-that-malware/
[3]: https://www.privacytools.io/webrtc.html
[4]: https://wiki.gnome.org/Projects/NetworkManager
