# FreeStyle Optium

Reverse engineered by [Diego Elio Pettenò](mailto:flameeyes@flameeyes.eu).

## Important device notes

The FreeStyle Optium device is a glucometer that can also read special β-ketones
testing strips. This will become clear in the protocol description.

## Cable

The Abbott FreeStyle Optium device uses a special cable, that connects
through the same port as the testing strips, and puts the device into
PC connection mode.

In PC connection mode, the screen will display the following:

    ]--PC

The cable itself, as provided by Abbott, uses a TIUSB3410 USB-to-Serial adapter
provided by Texas Instruments, similar to the TRS (stereo-jack) of previous
versions.

The strip-port adapter is fully suppored by Linux kernel 3.14 and
later. Previous kernels only support the TRS cable.

## Protocol

The communication protcol of this device is a serial, text-based protocol,
similar to previous FreeStyle devices, but not compatible.

### Serial port configuration

The serial port should be configured as such:

* 8 data bits;
* no parity bits;
* 1 stop bit;
* 19200 baud rate.

### Messages and commands

The protocol knows two type of messages: commands and responses.

    generic-message = command / response

Exchange is initiated by the host by sending one of the known commands. The
first command sent by the host is usually ignored by the device, so if a single
empty response is returned, the command should be retried.

The only command that accept parameters is `tim`.

    commands = ( "$colq" / "$xmem" / "tim" tim-parameters ) CR LF

Lines are generally terminated by the `CR LF` sequence. Some results sometimes
terminated by an okay sequence.

    command-succeeded = "CMD OK" CR LF

#### `$colq` command

The `$colq` command provides detailed information on the device:

    colq-response = "S/N:" HTAB serial-number CR LF
                    "Ver:" HTAB software-version HTAB glucose-unit CR LF
                    "Clock:" HTAB date-1-sec CR LF
                    "Market:" HTAB DIGIT HTAB DIGIT CR LF
                    "ROM:" HTAB DIGIT HTAB DIGIT HTAB DIGIT HTAB DIGIT CR LF
                    "Usage:" HTAB usage-count CR LF
                    command-succeded
    usage-count-1 = 1*DIGIT

    serial-number = 7( ALPHA / DIGIT ) "-" 5( ALPHA / DIGIT )
    software-version = 1*DIGIT "." 1*DIGIT

    glucose-unit = "MMOL" / ??? ; the output for mg/dL is not known

The `usage-count` is the number of readings available in the memory of the
device.

Serial number, software version, market and ROM output are really opaque
identifiers. The serial number is unique to the device (corresponds to the one
printed on its back) and can be used to match an user to their glucometer. All
these parameters are constant.

The `glucose-unit` of the device is reported together with the version. *Note:* as
I live in Ireland, and the unit is specified in mmol/L, I have no idea what the
string would be for glucometers in the rest of the world.

#### `$xmem` command

The `$xmem` command reports the content of the device's memory.

    xmem-response = CR LF
                    serial-number CR LF
                    software-version CR LF
                    date-2-sec CR LF
                    usage-count-2 CR LF
                    *(result CR LF)
                    checksum SP SP "END" CR LF
    usage-count-2 = 3DIGIT
    checksum = "0x" 4HEXDIG

The `usage-count-2` value matches `usage-count-1`, zero-padded to align at three
digits.

The `checksum` is a simple accumulation of the ASCII values of the characters,
up until the `0x` literal (excluded), including the `CR LF` terminations.

Each result is returned in its own line complete with the timestamp and some
(mostly opaque) metadata:

    result = result-value SP
             SP date-2-nosec
             SP "G" ; see note
             SP 0x00
    result-value = 3DIGIT / ( "HI" SP ) ; in mg/dL

The `G` literal is likely related to distinguish blood glucose results versus
β-ketones, unfortunately I have no such strip to test.

The `HI` literal signify a reading that was higher than the maximum reported
value of the device (value unknown.)

The result result values reported in numbers are always reported in mg/dL, even
if the glucometer is set to report mmol/L in its configuration. Conversion
happens at display time, both on the device and on the software.

#### `tim` command

Setting the time on the device is the only command that accept parameters, it is
also the only known command that does not start with the `$` character.

    tim-command = "tim" tim-parameters
    tim-parameters = "," tim-month "," tim-day "," tim-year
                     "," tim-hour "," tim-minute

    tim-response = command-succeeded ; not other output

The time on the device can only be set for full minutes, without including
seconds.

Years are represented in two-digits only, which are interpreted between 2000
and 2099. The device **can** keep track of time beyond that, as can be proven by
setting the time to 2099-12-31 23:59.

The device does not suffer from [Year 2038](https://en.wikipedia.org/wiki/Y2038)
problem.

#### Date formats

This device, similarly to previous Abbott devices, implements multiple date
formats, that are not interchangable:

    date-1-sec = month1 SP SP day SP year HTAB timesec
    date-2-sec = month2 SP day SP year SP timesec
    date-2-nosec = month2 SP day SP year SP time

    time = hour ":" minute
    timesec = time ":" second
    year = 4DIGIT
    day = 2DIGIT
    hour = 2DIGIT
    minute = 2DIGIT
    second = 2DIGIT

    month1 = "Jan" / "Feb" / "Mar" / "Apr" / "May" / "Jun" / "Jul" /
             "Aug" / "Sep" / "Oct" / "Nov" / "Dec"
    ; In this case, months are 4-characters strings, padded by space with the
    ; exception of June and July.
    month2 = "Jan " / "Feb " / "Mar " / "Apr " / "May " / "June" / "July" /
             "Aug " / "Sep " / "Oct " / "Nov " / "Dec "

The two formats used in the response for `$xmem` include the same idiosyncrasy
as previous devices, in which month names are truncated to three letters and
followed by a space character, with the exclusions of `June` and `July` which
are written in full. This behaviour is not present in the `$colq` response.

## References

* [TIUSB3410 datasheet](http://www.ti.com/lit/ds/symlink/tusb3410.pdf)
* [Alexander Schrijver's FreeStyle reverse engineering](http://www.flupzor.nl/protocol.html)