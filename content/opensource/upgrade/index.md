---
title: "Upgrade"
description: "Learn how to maintain and upgrade your Kerberos Open Source agents to a newer version."
lead: "Learn how to maintain and upgrade your Kerberos Open Source agents to a newer version."
date: 2020-10-06T08:49:31+00:00
lastmod: 2020-10-06T08:49:31+00:00
draft: false
images: []
menu:
  opensource:
    parent: "opensource"
weight: 207
toc: true
---

**_Kerberos Open Source is deprecated, please [use Kerberos Agent](/agent/first-things-first) instead._**

To upgrade your Kerberos agent to a new version you should follow the approach which fits your initial installation method. If you installed KiOS follow the KiOS upgrade procedure, if installed on Raspbian follow the Raspbian upgrade procedure, etc.

Please note that it might be possible that some new files are added or existing files were updated. To make sure everything works as expected, you should clear your browser cache.

## KiOS

If you installed KiOS, you can use the built-in upgrade method, `fwupdate`. The `fwupdate` command is a shell script which contains a couple of functions. For example it allows you to download, extract and flash a new version of KiOS to your SD card. The process is pretty straight forward.

First SSH or connect to KiOS first, and execute following command to see your current version.

```bash
fwupdate current
```

Next look for all available versions of KiOS.

```bash
fwupdate versions
```

Select the version to which you would like to upgrade, and run following command.

```bash
fwupdate upgrade <version>
```

KiOS will reboot, and your new version will be available.

## Raspbian

If you want to install a new version of the Kerberos agent for Raspbian, there is no automated versioning process available like KiOS. To perform an upgrade you'll need to follow the [traditional installation](/opensource/installation#raspbian) procedure of Raspbian.

## Docker

When a new release is available, a new Docker image will be available on the Docker hub. You can simply delete your existing container and image, and download it again.
