---
layout: post
title: Syncing multiple Pi-Holes
date: 2023-03-27 00:15 -0400
categories: homelab self-hosting, networking
tags: homelab blog docker dns pi-hole
---
## Summary

[Pi-Hole](https://pi-hole.net/) is a popular open-source network-level advertisement and internet tracker blocking application. It is designed to run on a single device, such as a Raspberry Pi, but many users may have multiple Pi-hole instances for redundancy and scalability purposes. To ensure that these instances are synchronized and have the same blocklists, whitelists, and DNS records, using [Orbital Sync](https://github.com/mattwebbio/orbital-sync) can be a convenient solution.

Orbital Sync is a tool that allows users to keep multiple Pi-hole instances in sync with each other. This tool relies on a primary Pi-hole instance to keep track of the things, and it pushes updates to the other Pi-holes. Here are the steps I took to use Orbital Sync in my own homelab:

## Steps

1. Install Orbital Sync on the primary Pi-hole instance: This can be done using docker or `node`. Docker is the recommended method, so I will be using that in this guide. Below is the `docker-compose` file I used to install Orbital Sync on my primary Pi-hole instance:

```yaml
version: '3'
services:
  orbital-sync:
    image: mattwebbio/orbital-sync:1
    container_name: orbital-sync
    environment:
      PRIMARY_HOST_BASE_URL: 'http://192.168.86.10'
      PRIMARY_HOST_PASSWORD: '${PRIMARY_PASSWORD}'
      SECONDARY_HOST_1_BASE_URL: 'http://192.168.86.12'
      SECONDARY_HOST_1_PASSWORD: '${SECONDARY_PASSWORD}'
      INTERVAL_MINUTES: 60
      NOTIFY_ON_SUCCESS: 'false'
      NOTIFY_ON_FAILURE: 'true'
      NOTIFY_VIA_SMTP: 'true'
      VERBOSE: 'true'
      TZ: 'America/New_York'
      SMTP_HOST: 'smtp.gmail.com'
      SMTP_PORT: 587
      SMTP_USER: '${EMAIL}'
      SMTP_PASSWORD: '${EMAIL_APP_PASSWORD}'
      SMTP_TO: '${EMAIL}'
```

The configuration is fairly straightforward but a few things to call out:

- I'm using password authentication for connecting to my Pi-Hole instances. These are the same passwords that you use to log into the admin interface.
- `INTERVAL_MINUTES` is set to 60 minutes. This means that Orbital Sync will check for updates every 60 minutes and push them to the secondary Pi-Hole instance if there are any.
- I'm using gmail to receive notifications (`SMTP_HOST`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_TO`). This is optional, but it can be useful to know if there are any issues with the synchronization.

2. Run the container: Once the `docker-compose` file is created, you can run the container using the following command: `docker-compose up -d`. This will start the container and it will begin checking for updates every 60 minutes. The output (`docker-compose logs -f`) will look something like this:

```bash
orbital-sync  | 3/27/2023, 1:38:10 PM: ➡️ Signing in to http://192.168.86.10/admin...
orbital-sync  | 3/27/2023, 1:38:10 PM: ✔️ Successfully signed in to http://192.168.86.10/admin!
orbital-sync  | 3/27/2023, 1:38:10 PM: ➡️ Downloading backup from http://192.168.86.10/admin...
orbital-sync  | 3/27/2023, 1:38:10 PM: ✔️ Backup from http://192.168.86.10/admin completed!
orbital-sync  | 3/27/2023, 1:38:10 PM: ➡️ Signing in to http://192.168.86.12/admin...
orbital-sync  | 3/27/2023, 1:38:10 PM: ✔️ Successfully signed in to http://192.168.86.12/admin!
orbital-sync  | 3/27/2023, 1:38:10 PM: ➡️ Uploading backup to http://192.168.86.12/admin...
orbital-sync  | 3/27/2023, 1:38:54 PM: ✔️ Backup uploaded to http://192.168.86.12/admin!
orbital-sync  | 3/27/2023, 1:38:54 PM: Result:
orbital-sync  | Processed adlist (7 entries)<br>
orbital-sync  | Processed adlist group assignments (7 entries)<br>
orbital-sync  | Processed blacklist (exact) (0 entries)<br>
orbital-sync  | Processed blacklist (regex) (0 entries)<br>
orbital-sync  | Processed client (0 entries)<br>
orbital-sync  | Processed client group assignments (0 entries)<br>
orbital-sync  | Processed local DNS records (66 entries)<br>
orbital-sync  | Processed local CNAME records (40 entries)<br>
orbital-sync  | Processed black-/whitelist group assignments (25 entries)<br>
orbital-sync  | Processed group (1 entry)<br>
orbital-sync  | Processed whitelist (exact) (18 entries)<br>
orbital-sync  | Processed whitelist (regex) (7 entries)<br>
orbital-sync  | OK
...
orbital-sync  |    [✓] Pi-hole blocking is enabled
orbital-sync  | 3/27/2023, 1:39:26 PM: ✔️ Success: 1/1 hosts synced.
orbital-sync  | 3/27/2023, 1:39:26 PM: Waiting 60 minutes...
```

If you run into any issues, the logs will provide more information about what went wrong. I ran into a few issues when I first set this up, but I was able to resolve them by checking the logs.

3. Verify the synchronization: Finally, you can verify that the Pi-hole instances are synchronized by checking the blocklists or DNS records on each instance. If everything is working properly, the values should be identical on each Pi-hole instance.

## Conclusion

In conclusion, using Orbital Sync can be a simple and effective way to keep multiple Pi-hole instances in sync with each other. I found the setup process to be much easier than that of similar tools. By following these steps, you can ensure that all of your Pi-hole instances have the same values for blocklists, whitelists, and DNS records. That way if your primary Pi-hole instance goes down, you can still use the secondary Pi-hole instance without any issues.
