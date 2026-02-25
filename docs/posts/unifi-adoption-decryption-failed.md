---
authors: [stimmerman]
comments: true
categories:
  - Unifi
date: 2026-02-25
tags:
  - unifi
  - networking
  - mongodb
---
# Unifi adoption failed: decryption error after improper device removal

Unable to adopt a Unifi AP that was previously visible in the controller. The device would not show up for adoption again despite being reset multiple times.

<!-- more -->

## Diagnosis

Checked the controller's `server.log` and found these decryption failures matching the AP's MAC address:

```
[2026-02-24T22:14:00,564+01:00] <inform-22> WARN  inform - dev[1C-6A-1B-EB-74-24] inform decryption failed with defaultAuthKey=true, from xxx.xxx.xxx.xxx:63009 GeneralSecurityException
[2026-02-24T22:14:00,564+01:00] <inform-22> WARN  inform - dev[1C-6A-1B-EB-74-24] unable to decrypt inform, from xxx.xxx.xxx.xxx:63009
```

## Root Cause

The AP was previously adopted but not properly removed from the controller. The controller retained the device entry with custom encryption keys, while the AP was attempting to communicate using default keys after a factory reset.

## Solution

1. You might have to expose the MongoDB port if you're using a typical docker compose stack with a unifi controller and mongo db.
```
  unifi-db:
    image: docker.io/mongo:7.0.21
    container_name: unifi-db
    ports:
      - 27017:27017
    volumes:
      - unifi-db:/data/db
      - ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
    restart: unless-stopped
```
2. Access the Unifi controller's MongoDB database (in my case called 'unifi'), I used DataPlus to connect over SSH to it.
3. In the `device` collection/table, search for the AP's MAC address in the `mac` column.
4. Delete that entry.
5. Reboot the Unifi controller.
6. Factory reset the AP.
7. Attempt adoption again.

The AP should now appear in the adoption list and adopt successfully using default encryption keys.
