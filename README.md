# Miniupnpd <=v2.1 read out-of-bounds vulnerability (PoC)

* This vulnerability has been fixed within miniupnpd's master branch (https://github.com/miniupnp/miniupnp/commit/bec6ccec63cadc95655721bc0e1dd49dac759d94).
* The vulnerability is triggered when sending a **SUBSCRIBE** request with a callback uri `obj->path` that is greater than 526 bytes.
* The root cause is due to the lack of validating the return value of `snprintf()` since `snprintf()` returns the value of how many bytes it *could* of copied, not how many bytes it did copy.
* As of Jan-25-2019 the PoC within this repro has been successfully tested against Google Wifi.
  * Other devices that utilize `miniupnpd` may be vulnerable as well.

## Root Cause (upnpevents.c)
```
static void upnp_event_prepare(struct upnp_event_notify * obj)
{

	obj->buffersize = 1024; /* Static Buffer Size */
	obj->buffer = malloc(obj->buffersize);
	[...]
	obj->tosend = snprintf(obj->buffer, obj->buffersize, notifymsg,
	                       obj->path, obj->addrstr, obj->portstr, l+2,
	                       obj->sub->uuid, obj->sub->seq,
	                       l, xml);
	obj->state = ESending;

static void upnp_event_send(struct upnp_event_notify * obj)
{
	int i;
	i = send(obj->s, obj->buffer + obj->sent, obj->tosend - obj->sent, 0);
```

**Man Page Entry for snprintf()**
```
RETURN VALUE

Upon successful return, functions return the number of characters printed 
(excluding the null byte used to end output to strings).

The functions snprintf() and vsnprintf() do not write more than size bytes 
(including the terminating  null byte ('\0')).  If the output was truncated 
due to this limit, then the return value is the number of characters 
(excluding the terminating null byte) which would have been written to the 
final string if enough space had been available. Thus, a return value of size 
or more means that the output was truncated.
```

## Usage
```
usage: miniupnpd_poc.py [-h] [--callback_ip CALLBACK_IP]
                        [--callback_port CALLBACK_PORT] [--timeout TIMEOUT]
                        [--leak_amount LEAK_AMOUNT]
                        target_ip target_port

Miniupnpd <= v2.1 read out-of-bounds vulnerability

positional arguments:
  target_ip             IP address of vulnerable device.
  target_port           Target Port.

optional arguments:
  -h, --help            show this help message and exit
  --callback_ip CALLBACK_IP
                        Local IP address for httpd listener. (default: None)
  --callback_port CALLBACK_PORT
                        Local port for httpd listener. (default: None)
  --timeout TIMEOUT     Timeout for http requests (seconds). (default: 5)
  --leak_amount LEAK_AMOUNT
                        Amount of arbitrary heap data to leak (in Kb).
                        (default: 1)
```

## Video
[![asciicast](https://asciinema.org/a/6PYTXJjkiNWx20RH5cWuZS1ie.svg)](https://asciinema.org/a/6PYTXJjkiNWx20RH5cWuZS1ie)

- 0wl