PassPort
========

<p align="center">
<img src="https://github.com/c-skills/passport/blob/master/passport.jpg"/>
</p>

Forwarding TCP ports through [Passkey](https://fidoalliance.org/passkeys/) servers to bypass censorship.

*Disclaimer: This writeup and demo code is for educational purposes only! It serves to research the new
FIDO CTAP and related standards and risk assess this new technology. I will take no responsibility for
any of your actions. It is assumed and strongly recommended that you consult the EULA and any and all
regulations that might apply in your area before making any tests of your own.
Do not send me requests for improvements or ask for any help in "making it work".*


Introduction
------------

Recently, the new FIDO standards that incorporate WebAuthn and [CTAP](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#sctn-hybrid) to form *Passkeys* were
widely marketed as *the* password replacement. A quite new chapter about *Hybrid Transports* allows
to use QR codes with the authenticator (in this case most likely your smart phone) to authenticate
against a service in that authenticator and client platform (in this case most likely your browser)
open a tunnel to send PDUs back and forth in a similar way that they would travel via USB or NFC.


A public CTAP tunnel server?
----------------------------

Yes. A high reputation server IP that allows you to tunnel binary data and it speaks WebSockets as per standard.

The whole idea is actually very simple and best understood with a graphic:

```

                    Outside World [ "Free" Internet ]



   SSH Server                                                   "google.com"

[ 127.0.0.1:22 ] <---- passport -tconnect 127.0.0.1 22 -----> | cable.ua5v.com  | <--+
                                                                                     |
                                                                                     |
                                                                                     |
                                                                                     |
--------------------/// The Great Berlin Wall \\\------------------------------------o-----
                                                                                     |
                                                                                     |
                    Inside World [ "Censored Network" ]                              |
                                                                                     |
        local port                                                                   |
                                                                                     |
     [ 127.0.0.1:1234 ] <----> passport -tnew   -------------------------------------+

```

For my research I will focus on the Google server, which is named `cable.ua5v.com`
as per standard. The Apple server is called `cable.auth.com` and I guess it works
in a similar fashion, but I did not test it.

The way to create a new tunnel connection is vendor dependent, but it is easily found out
by reading the chromium sources. The way to connect to such a newly created tunnel is per
standard.

I already "reversed" all the protocol details and it can be found in the `passport`
script. If you can read C++ code, its pretty easy.

As can be seen in the picture, the idea is a simple proxy that opens a local port
and forwards it across the WebSocket connection to an outside SSH server. So you
can SSH from behind a censorship to a "free" SSH server and then use it to
forward web traffic as usual.

This ofcorse requires the censor to allow connections to `cable.ua5v.com`. Why
should they do that?

* The IP is in Googles IP-range, the same IP you get when you browse "google.com"
  or lets just say "google.hk". Yes, if the entire Google net-range is just dropped,
  this will not work, unless you have other existing vendors which are white listed ...

* Domain fronting works fine. This means you can connect to the IP that is reachable
  and put any legit looking SNI into the TLS client hello, e.g. "google.hk" so that
  censorship observers can not distinguish a web-search on a presumed self-censored
  site from a tunnel connections. Within the request, the `Host: cable.ua5v.com` is found.



Some more details
-----------------

There are two modes of operation. Both are artificially throtteld so to demo that the idea
works but to not consume real bandwidth or server power.

In manual mode via `-new` you have to pass the obtained routing-ID (check the standard) to the
client *Outside* so he can `-connect` to the *Inside* tunnel endpoint.

In automatic mode, `-tconnect` will obtain all possible routing-IDs (do not ask me why they
are predictable for a time-frame of like 24h and have 1/6 chance to succeed), keep it
and try until success without needing to establish a second channel between *Inside* and *Outside*.

As the tunnel server shuts down both connections after 60s, automatic mode will reconnect
and try the predicatble IDs until a new tunnel is established, *leaving the existing SSH
connection alive and working.*

For testing, when a `-tnew` is started, it expects the connection on
localhost at the given port first, which it then forwards across the
WebSocket once that the other end succeeded with their `-tconnect`
at the same time (obtaining the routing IDs can take ~10s).
Make sure you succeed within OpenSSH timeouts and all necessary terminals.

It is required that *Inside* and *Outside* have synchronized clocks
(Epoch) in hour-granularity when automatic mode is used, in order to derive
equal tunnel IDs.


Was there any Bug Bounty?
-------------------------

No. I did not even bother to ask for it, as IMHO there is actually no bug. Everything works
by the standard. I cannot imagine what could be different when the standard explicitely allows
or even endorses WebSocket connections through high reputation IP ranges. I am thankful
this Hybrid Transport was added and my reason to publish it was to make it more widely
known. I hope that more large vendors are added in future so to make it impossible for
censorship to block it all without also disabling their own sessions.


