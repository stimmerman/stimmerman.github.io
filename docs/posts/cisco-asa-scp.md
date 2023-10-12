---
authors: [stimmerman]
comments: true
categories:
  - Cisco
  - ASA
date: 2023-10-12
tags:
  - cisco
  - asa
  - scp
---
# SCP files to and from Cisco ASA
I needed to copy some file over SSH to a Cisco ASA appliance, but ran in the SCP error on the bottom of the page which was not an easy to find solution.
<!-- more -->

## Copy a file to the ASA
Replace or remove `disk0` with another directory if you need to.
```
scp your-file-here.txt username@192.168.1.1:disk0:your-file-here.txt
```

## Copy a file from the ASA
Replace or remove `disk0` with another directory if you need to.
```
scp username@192.168.1.1:disk0:your-file-here.txt ~/your-file-here.txt
```

!!! warning "Original SCP protocol"
    
    If your SCP command fails with
    ```bash
    subsystem request failed on channel 0
    scp: Connection closed
    ```
    Then you need to run the SCP commands with the `-O` flag added. To force the use of the original SCP protocol instead of SFTP.
    ```
    scp -O your-file-here.txt username@192.168.1.1:disk0:your-file-here.txt
    ```
    From the linux man page:
    > -O Use the legacy SCP protocol for file transfers instead of the SFTP protocol.  Forcing the use of the SCP protocol may be necessary for servers that do not implement SFTP, for backwards-compatibility for particular filename wildcard patterns and for expanding paths with a ‘~’ prefix for older SFTP servers.

