---
layout: base
title: Improv via BLE
description: All the implementation details necessary to make your own client and service implementation.
---

This is the description of the Improv Wi-Fi protocol using Bluetooth Low Energy.

The protocol has two actors: the Improv service running on the gadget and the Improv client.

The Improv service will broadcast its presence via Bluetooth LE and receives Wi-Fi credentials from the client.

The Improv client detects the service via Bluetooth LE and will offer the user to send Wi-Fi credentials.

## Improv Service

The Improv service runs on a gadget that needs to connect to the internet but has no credentials or is unable to establish a connection.

The Improv service can optionally require physical authorization to allow pairing, like pressing a button. It is up to the gadget to decide if and what interaction to pick. A gadget that does not require authorization should start in the "authorized" state.

If an Improv service has been authorized by a user interaction, the authorization should be automatically revoked after a timeout. This timeout is up to the gadget but we suggest 1 minute.

A user is able to send Wi-Fi credentials to an authorized service. The gadget will attempt to connect to the specified wireless network. If the connection is successful, the state changes to "provisioned", the gadget can optionally return a URL to the client to finish onboarding and the Improv service is stopped.

If the gadget is unable to connect an error is returned. If the gadget required authorization, the authorization reset timeout should start over.

![Improv State machine](/images/improv-states.svg)

The client is able to send an `identify` to the Improv service if it is in the states "Require Authorization" and "Authorized". When received, and enabled, the gadget will identify itself, like playing a sound or flashing a light. It is up to the gadget to decide if and what interaction to pick.

The client is able to send a `device info` to the Improv service if it is in the states "Require Authorization" and "Authorized". When received, and supported, the gadget will return the device information in the RPC response characteristic.

All strings are assumed to be UTF-8 encoded.

## Revision history

- 1.0 - Initial release
- 2.0 - Added Service Data `4677`
- 2.1 - Added Device Info RPC command
- 2.2 - Added Scan Wifi RPC command
- 2.3 - Added Hostname RPC command
- 2.4 - Added Device Name RPC command
- 2.5 - Added Secure Provisioning (Setup Secure Session + Secure Envelope RPC commands, optical SAS)

## GATT Services

### Characteristic: Capabilities

Characteristic UUID: `00467768-6228-2272-4663-277478268005`

This characteristic has binary encoded byte(s) of the device’s capabilities.

| Bit (LSB) | Capability                                        |
|-----------|---------------------------------------------------|
| `0`       | 1 if the device supports the identify command.    |
| `1`       | 1 if the device supports the device info command. |
| `2`       | 1 if the device supports the scan wifi command.   |
| `3`       | 1 if the device supports the hostname command.    |
| `4`       | 1 if the device supports secure provisioning.     |


### Characteristic: Current State

Characteristic UUID: `00467768-6228-2272-4663-277478268001`

This characteristic will hold the current status of the provisioning service and will write and notify any listening clients for instant feedback.

| Value  | State                  | Purpose                                          |
| ------ | ---------------------- | ------------------------------------------------ |
| `0x01` | Authorization Required | Awaiting authorization via physical interaction. |
| `0x02` | Authorized             | Ready to accept credentials.                     |
| `0x03` | Provisioning           | Credentials received, attempt to connect.        |
| `0x04` | Provisioned            | Connection successful.                           |

### Characteristic: Error state

Characteristic UUID: `00467768-6228-2272-4663-277478268002`

This characteristic will hold the current error of the provisioning service and will write and notify any listening clients for instant feedback.

| Value  | State               | Purpose                                                                                 |
|--------|---------------------|-----------------------------------------------------------------------------------------|
| `0x00` | No error            | This shows there is no current error state.                                             |
| `0x01` | Invalid RPC packet  | RPC packet was malformed/invalid.                                                       |
| `0x02` | Unknown RPC command | The command sent is unknown.                                                            |
| `0x03` | Unable to connect   | The credentials have been received and an attempt to connect to the network has failed. |
| `0x04` | Not Authorized      | Credentials were sent via RPC but the Improv service is not authorized.                 |
| `0x05` | Bad Hostname        | The hostname provided was not valid or acceptable by the device.                        |
| `0x06` | Encryption Required     | The device requires a secure session; plaintext credentials were refused.               |
| `0x07` | Secure Handshake Failed | The secure session setup failed: bad public key, authentication failure, or timeout.    |
| `0xFF` | Unknown Error       |

### Characteristic: RPC Command

Characteristic UUID: `00467768-6228-2272-4663-277478268003`

This characteristic is where the client can write data to call the RPC service.

Note: if the combined payload is over 20 bytes, it will require multiple BLE packets to transfer the data. Make sure that your code deals with this.

| Byte  | Description                                           |
| ----- | ----------------------------------------------------- |
| 1     | Command (see below)                                   |
| 2     | Data length                                           |
| 3...X | Data                                                  |
| X + 3 | Checksum - A simple sum checksum keeping only the LSB |

#### RPC Command: Send Wi-Fi settings

Submit Wi-Fi credentials to the Improv Service to attempt to connect to.

Requires the Improv service to be authorized.

Command ID: `0x01`

| Byte | Description     |
| ---- | --------------- |
| 01   | command         |
| xx   | data length     |
| yy   | ssid length     |
|      | ssid bytes      |
| zz   | password length |
|      | password bytes  |
| CS   | checksum        |

Example: SSID = MyWirelessAP, Password = mysecurepassword

```
01 1E 0C {MyWirelessAP} 10 {mysecurepassword} CS
```

This command will generate an RPC result. The first entry in the list is an URL to redirect the user to. If there is no URL, omit the entry or add an empty string.

#### RPC Command: Identify

What a device actually does when an identify command is received is up to that specific device, but the user should be able to visually or audibly identify the device.

Command ID: `0x02`

Does not require the Improv service to be authorized.

Should only be sent if the capability characteristic indicates that identify is supported.

| Byte | Description            |
| ---- | ---------------------- |
| 02   | command                |
| 00   | 0 data bytes / no data |
| CS   | checksum               |

This command has no RPC result.

#### RPC Command: Device Info

Sends a request for the device to send information about itself.

Command ID: `0x03`

Does not require the Improv service to be authorized.

Should only be sent if the capability characteristic indicates that device info is supported.

| Byte | Description            |
|------|------------------------|
| 03   | command (`0x03`)       |
| 00   | 0 data bytes / no data |
| CS   | checksum               |

This command will generate an RPC result. There will be at least 4 entries in the list response.

Order of strings: Firmware name, firmware version, hardware chip/variant, device name. Optionally, the OS name and OS
version can be appended if applicable and different from the firmware name/version.

Example without OS Name: `ESPHome`, `2021.11.0`, `esp32-s3-devkitc-1/esp32-s3`, `Temperature Monitor`.

Example with OS Name: `Bluetooth Proxy`, `v1.0.0`, `denky_d4/esp32`, `My Bluetooth Proxy`, `ESPHome`, `2025.12.2`.

### RPC Command: Request scanned Wi-Fi networks

Sends a request for the device to send the Wi-Fi networks it sees.

Command ID: `0x04`

| Byte | Description            |
|------|------------------------|
| 04   | command (`0x04`)       |
| 00   | 0 data bytes / no data |
| CS   | checksum               |

This command will trigger one RPC Response which will contain a multiple of 3 strings where the first contains the SSID,
the second the RSSI and the third the authentication type of either WEP, WPA, WPA2, WPA2 EAP, WPA3, WAPI or NO. If 
multiple authentication types are supported they should be separated by a forward slash `/`.

Order of strings: Wi-Fi SSID 1, RSSI 1, Auth type 1, Wi-Fi SSID 2, RSSI 2, Auth type 2, ...

Example: `MyWirelessNetwork`, `-60`, `WPA2`, `MyOtherWirelessNetwork`, `-52`, `WPA/WPA2`,...

A response with no strings means no SSID was found.

### RPC Command: Get/Set Hostname

Sends a request for the device to either get or set its hostname.
This operation is only available while the device is Authorized.  Sending the command with no data will return
the current hostname in the response. Sending the command with data will set the hostname to the data and also
return the updated hostname in the response. Setting this property requires the device to be in an 'Authorized' state.

Hostnames must conform to [RFC 1123](https://datatracker.ietf.org/doc/html/rfc1123) and can contain only letters,
numbers and hyphens with a length of up to 255 characters.  Error code `0x05` will be returned if the hostname provided is not acceptable.

Command ID: `0x05`

Get Hostname:

| Byte | Description            |
|------|------------------------|
| 05   | command (`0x05`)       |
| 00   | 0 data bytes / no data |
| CS   | checksum               |

Set Hostname:

| Byte | Description        |
|------|--------------------|
| 05   | command (`0x05`)   |
| XX   | length of hostname |
|      | bytes of hostname  |
| CS   | checksum           |

This command will trigger one RPC Response which will contain the hostname of the device. Setting this
property should reset the authorization timeout.

### RPC Command: Get/Set Device Name

Sends a request for the device to either get or set its name. This could mean different things depending on the device
manufacturer.  It may alter the default "hostname" or not. If setting both this property and hostname, it is recommended
to set the device name first then the hostname. Getting this property should return the same value as the Device Info's 
"Device Name" (4th) property. Setting this property requires the device to be in an 'Authorized' state.

Command ID: `0x06`

Get Device Name:

| Byte | Description            |
|------|------------------------|
| 06   | command (`0x06`)       |
| 00   | 0 data bytes / no data |
| CS   | checksum               |

Set Device Name:

| Byte | Description                    |
|-----|--------------------------------|
| 06  | command (`0x06`)               |
| XX  | length of device name in bytes |
|     | bytes of device name           |
| CS  | checksum                       |

This command will trigger one RPC Response which will contain the Device Name of the device. Setting this
property should reset the authorization timeout.


#### RPC Command: Setup Secure Session

Starts an authenticated key exchange so that the Wi-Fi credentials (and any other RPC) can be exchanged encrypted and protected against a man-in-the-middle. See [Secure Provisioning](#secure-provisioning) for the full process.

Command ID: `0x07`

Should only be sent if the capability characteristic indicates that secure provisioning is supported (bit 4).

Request:

| Byte  | Description                                     |
| ----- | ----------------------------------------------- |
| 07    | command (`0x07`)                                |
| 22    | data length (34)                                |
|       | client X25519 public key (32 bytes)             |
|       | requested optical bit rate (2 bytes, u16 LE)    |
| CS    | checksum                                        |

This command generates an RPC result with the device's ephemeral public key and the negotiated optical bit rate:

| Byte  | Description                                     |
| ----- | ----------------------------------------------- |
| 07    | command (`0x07`)                                |
| 22    | data length (34)                                |
|       | device X25519 public key (32 bytes)             |
|       | negotiated optical bit rate (2 bytes, u16 LE)   |
| CS    | checksum                                        |

The result of this command is a raw (unencrypted) payload: it is the only secure-provisioning message not wrapped in a Secure Envelope, because it establishes the key. The public keys are safe to exchange in the clear.

#### RPC Command: Secure Envelope

Carries an encrypted inner RPC command (for example Send Wi-Fi settings) once a secure session has been established with Setup Secure Session.

Command ID: `0x08`

| Byte  | Description                                       |
| ----- | ------------------------------------------------ |
| 08    | command (`0x08`)                                 |
| xx    | data length                                      |
|       | message counter (8 bytes, u64 LE)                |
|       | ciphertext (AEAD-encrypted inner RPC packet)     |
|       | authentication tag (16 bytes)                    |
| CS    | checksum                                         |

The plaintext is an ordinary RPC packet (command / length / data / checksum), so any command may be secured. The device decrypts it and processes it exactly as if it had been received in the clear; the RPC result is returned wrapped in a Secure Envelope in the same way. See [Secure Provisioning](#secure-provisioning) for the envelope and key details.


### Characteristic: RPC Result

Characteristic UUID: `00467768-6228-2272-4663-277478268004`

This characteristic is where the client can read results from the RPC service if it has a result. Results are returned as a list of strings. An empty list is allowed.

| Byte      | Description                                           |
| --------- | ----------------------------------------------------- |
| 1         | Command (see below)                                   |
| 2         | Data length                                           |
| 3         | Length of string 1                                    |
| 4...X     | String 1                                              |
| X         | Length of string 2                                    |
| X...Y     | String 2                                              |
| ...       | etc                                                   |
| last byte | Checksum - A simple sum checksum keeping only the LSB |

## Secure Provisioning

Secure provisioning is an optional extension (capability bit 4) that adds confidentiality and man-in-the-middle (MITM) protection to the credential handoff. In the base protocol the Wi-Fi credentials are written in the clear, so a passive listener can capture the password and an active attacker can man-in-the-middle the exchange. This extension leaves the base protocol intact and works over Web Bluetooth (it does not use BLE pairing).

It is authenticated Diffie-Hellman: an ephemeral X25519 exchange provides confidentiality, and a Short Authentication String (SAS) transmitted out-of-band over the device's LED (read by the client's camera) provides MITM protection. It is structurally the same idea as BLE numeric comparison, performed at the application layer.

### Overview

1. (Optional) the user physically authorizes the device, as in the base protocol. The SAS is only emitted once the device is Authorized.
2. The client sends Setup Secure Session (`0x07`) with its ephemeral X25519 public key and a requested optical bit rate. The device replies with its own ephemeral public key and the negotiated bit rate.
3. Both sides derive the session key and the SAS (below). The device begins flashing the SAS on its LED, continuously, at the negotiated bit rate.
4. The client reads the SAS with its camera and compares it with the value it computed. On mismatch it aborts and sends nothing. On match it sends the Wi-Fi credentials inside a Secure Envelope (`0x08`).
5. The device stops flashing, decrypts the envelope, and continues exactly as the base protocol (Provisioning, then Provisioned with an optional redirect URL, returned in a Secure Envelope).

### Cryptographic primitives

| Purpose             | Primitive                                    |
| ------------------- | -------------------------------------------- |
| Key agreement       | X25519 (RFC 7748), 32-byte keys              |
| Hash                | SHA-256                                       |
| Key derivation      | HKDF-SHA-256 (RFC 5869)                       |
| Authenticated enc.  | ChaCha20-Poly1305 (RFC 8439), 256-bit key    |

Keys are 32 bytes encoded per RFC 7748. Multi-byte integers are little-endian.

### Key derivation

Let `c_pub` / `d_pub` be the client and device public keys and `ss` the X25519 shared secret (identical on both sides). Define:

```
transcript = c_pub (32 bytes) || d_pub (32 bytes)

K   = HKDF-SHA-256(salt = "improv-ble-secure-v1", ikm = ss,
                   info = 0x01 || transcript, L = 32)

SAS = SHA-256("improv-ble-secure-v1-sas" || transcript || ss)[0..3]   (32 bits)
```

`K` is the Secure Envelope key. `SAS` is the 32-bit value flashed on the LED and verified by the client. A man-in-the-middle holds different public keys with each side, so the SAS it produces with the device differs from the one the client computes; the client's comparison fails and it aborts before releasing credentials. 32 bits is sufficient because the attacker gets a single online attempt (it must commit to its keys before the SAS is revealed): a forced collision succeeds with probability about 2^-32 per provisioning attempt.

### Secure Envelope (command 0x08)

The `data` field of a Secure Envelope is:

```
counter    : u64 little-endian (8 bytes)   per-direction message counter, from 0
ciphertext : N bytes                        AEAD ciphertext of an inner RPC packet
tag        : 16 bytes                        Poly1305 authentication tag
```

AEAD parameters:

```
key       = K
nonce     = dir (1 byte) || 0x00 0x00 0x00 || counter (u64 LE)   (12 bytes)
            dir = 0x00 client to device, 0x01 device to client
aad       = "improv-ble-secure-v1" || dir
plaintext = an inner RPC packet (command || length || data || checksum)
```

Each direction keeps its own counter, starting at 0 and incremented per envelope; `dir` plus the per-direction counter ensures the nonce is never reused under `K`. A receiver MUST reject an envelope whose counter does not strictly increase for its direction, and MUST reject any envelope that fails authentication (error `0x07`). A successful decryption is itself proof that both sides hold the same `K`.

### Optical channel - Profile 1

The device transmits the 32-bit SAS by modulating its LED at the negotiated bit rate `R`; the client decodes it with its camera. Profile 1 is mandatory to implement.

- Modulation: NRZ on-off keying (LED on = 1, off = 0), one bit per symbol period `1/R`. Because the rate is negotiated the receiver knows it and recovers phase from the preamble, so Manchester self-clocking is not needed.
- Bit-rate negotiation: the client requests `floor(max_camera_fps / 3)` bits per second (three camera samples per symbol), defaulting to 40 bps (about 120 fps) if it cannot determine its camera rate. The device replies with `min(requested, device_max)`, capping at what its LED can modulate. Both clamp to 5-255 bps. The rate is a transport hint only and is not part of the transcript, key or SAS.
- The device SHOULD hold its symbol clock within 2%.
- Bit order is most-significant-bit first.
- Frame (transmitted repeatedly until the Secure Envelope arrives):

```
preamble : 0xAA 0xAA   trains symbol timing / phase
sfd      : 0x7E        start-of-frame delimiter
version  : 0x01        optical profile / SAS format version
sas      : SAS         32 bits
crc      : CRC-8       poly 0x07, over version || sas
```

The client locks phase on the preamble, waits for the SFD, checks `version` and `crc`, and SHOULD require two consecutive identical valid frames before comparing the SAS. At 40 bps the 72-bit frame takes about 1.8 s; at 5 bps about 14 s.

### Downgrade protection

The capability byte is read over the unauthenticated link, so an active attacker could hide the secure capability to force plaintext. Therefore:

- A client that requires security MUST offer a "secure only" mode and MUST NOT send plaintext credentials in that mode, regardless of the advertised capability.
- A device MAY be configured to require security: it then refuses the plaintext Send Wi-Fi settings command with error `0x06` (Encryption Required) and accepts credentials only inside a Secure Envelope.

### Security considerations

- Both sides MUST use fresh ephemeral keys per session (forward secrecy).
- The SAS authenticates the device-to-client direction; the client MUST abort on SAS mismatch before sending the Secure Envelope. The device gains assurance about the client from the physical authorization and from a successful envelope decryption.
- The device SHOULD discard the session and stop flashing after a timeout (suggested 60 s) if no valid envelope arrives.


## Bluetooth LE Advertisement

The device MUST advertise the Service UUID.

Service UUID: `00467768-6228-2272-4663-277478268000`

With version 2.1 of the specification:

- The Service Data and Service UUID MUST be advertised periodically and when the state changes.
- The Service Data and Service UUID MUST be in the same advertisement.
- The Service Data and Service UUID MUST NOT be in the scan response or require active scans.
- If the device cannot fit all of its advertising data in 31 bytes, it should cycle between advertising data.

### Service Data format

Service Data UUID: `4677` (`00004677-0000-1000-8000-00805f9b34fb`)

| Byte      | Description                                           |
| --------- | ----------------------------------------------------- |
| 1         | Current state                                         |
| 2         | Capabilities                                          |
| 3         | 0 (RESERVED)                                          |
| 4         | 0 (RESERVED)                                          |
| 5         | 0 (RESERVED)                                          |
| 6         | 0 (RESERVED)                                          |
