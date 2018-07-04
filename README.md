# Particle OTA problems with 3rd Party SIM

Here, I am attempting to demonstrate the failure of OTA updates using a 3rd party SIM card.

With all tests I have tried, OTA updating works correctly with a Particle SIM card.

## setup
1. Have an Electron running and connected to the cloud (using a 3rd party SIM card).
  - Ideally, this is running some code with "LOG_LEVEL_TRACE" enabled.

> For our tests, we used:
* SARA-G350 (2G Electron)
* 2 different SIMs (eseye, and geosim) with same results
* system firmware 0.6.4

2. have particle cli set up

3. clone this repo and `cd`

4. In a terminal window (different from the one you run commands in), `subscribe` to the device's events.

5. compile code like this:

```sh
particle compile electron --target 0.6.4 ./tinker_00.ino --saveTo tinker_00.bin
particle compile electron --target 0.6.4 ./tinker_00.ino --saveTo tinker_01.bin
```

6. flash tinker_01.bin to device *via --serial*
  - Start with this because it has the good Serial logging output.

## Test

### This Succeeds

Run this:

```sh
paricle flash [deviceName] tinker_00.bin
```

In the `subscribe` window, you should see output like:

```json
{"name":"spark/flash/status","data":"started ","ttl":60,"published_at":"2018-07-03T23:39:50.517Z","coreid":"123"}
{"name":"spark/flash/status","data":"success ","ttl":60,"published_at":"2018-07-03T23:40:06.822Z","coreid":"123"}
```

**IMPORTANT**
Now, to put it back so you have TRACE level logging, flash tinker_01.bin again *via --serial*.

### This Fails

Run this:

```sh
paricle flash [deviceName] tinker_01.bin
```

In the `subscribe` window, you should see output like:

```json
{"name":"spark/flash/status","data":"started ","ttl":60,"published_at":"2018-07-03T23:52:03.337Z","coreid":"123"}
{"name":"spark/device/last_reset","data":"update_timeout","ttl":60,"published_at":"2018-07-03T23:53:52.038Z","coreid":"123"}
```

### Theory

#### File Size
I believe this has to do with file size. Note that the `tinker_01.bin` is significantly bigger than `tinker_00.bin` (running `du -h tinker_00.bin` gives me 8KB, and on `tinker_01.bin` gives me 16KB).

Why do I think this matters?

#### idx=what?

Crucially, I noticed that when flashing tinker_00.bin, we get Serial output that includes:

```
0075261162 [comm] TRACE: starting file length 7820 chunks 16 chunk_size 512
```

So it's going to transfer 16 chunks of 512 bytes each, adding up to 8KB. Seems cool.

So it runs (with one little hiccup at chunk idx=15, which seems to right itself) until it sends the whole file (last chunk is idx=15).

(See log output at logs/output_tinker_01_to_tinker_00.log)


**However,** in the test where we flash tinker_01 to tinker_01, (the 16 kb file), it starts like this:

```
0000033003 [comm] TRACE: starting file length 14084 chunks 28 chunk_size 512
```

So, it's 28 chunks. But then as it begins downloading, it gets to a certain point and then hangs:

```
0000037059 [system] TRACE: received 557
0000037060 [comm] TRACE: chunk
0000037061 [comm] TRACE: chunk idx=15 crc=1 fast=1 updating=1
0000037063 [comm.protocol] INFO: rcv'd message type=7

```

### Conclusion

¯\_(ツ)_/¯


## To do:
* [ ] Try this with U260
