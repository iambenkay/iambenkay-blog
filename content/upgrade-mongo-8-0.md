+++
title = "My hackaround for upgrading MongoDB from 7.x to 8.0"
slug = "upgrade-from-mongo-7-x-to-8-0"
date = 2025-12-07
description = "Recently I had to upgrade MongoDB from 7.x to 8.0 to use some of the new features including bulk writes (the Rust SDK does not support this feature for 7.x versions yet). I tried to follow the official migration documentation for migrating an existing replica"

[extra]
header_img = "/images/header-mongo.jpg"
header_alt = "MongoDB"
section = "Programming"
word_count = 779
author = "Benjamin Chibuzor-Orie"
tags = ["MongoDB", "Upgrade"]
+++
Recently I had to upgrade **MongoDB** from `7.x` to `8.0` to use some of the new features including bulk writes (the Rust SDK does not support this feature for `7.x` versions yet). I tried to follow the official migration documentation for migrating an existing replica set to 8.0 but I encountered a major issue that I want to share with you. Ultimately we will discuss my workaround for this.

## Background
MongoDB has since made replica sets the recommended way to configure your **MongoDB** deployments even if it is just one instance. Configuring your database in this way ensures that your data is always available and protected from hardware failures thanks to the replication. This seemingly innocuous configuration which pretends like it is here to save you from the risk of data loss may also be the reason you are unable to upgrade your mongo deployment. I made sure to follow the documentation for migrating replica sets step by step until the end, the only unique thing about my scenario is that it is still a single node replica set.

Something I would advice is to make sure that you have a backup of your data before you start the migration process. This is especially important if you are running a single node replica set as it is the only node that contains all your data. If something goes wrong during the migration process, you will lose all your data.

The actual issue I encountered was after stepping down the node from primary and swapping the binaries to 8.0, the node would not start up. The error message in the container logs for mongo looked like WiredTiger related issues and it denoted an incompatibility or corruption of some sort.

## Solution
I found two ways to fix this problem, both of which involving transferring the data from the old node to a new node, just that one is done manually by you while the other is done automatically by the replica set configuration.

### Manual Transfer
In this case you will need to transfer the data from the `7.x` instance of the DB to a new `8.0` instance.
1. Create a new `8.0` instance of MongoDB.
2. Download **mongo** cli tools (specifically `mongodump` and `mongorestore`) for your operating system.
3. Use `mongodump` to dump the data from the `7.x` instance.
    Here is the command for dumping the data from the `7.x` instance:
    ```bash
    mongodump --uri "$MONGODB_URI" --gzip
    ```
    This will create a dump of your database in a folder called `dump` in the same directory. We used `--gzip` to compress the bson files
4. Use `mongorestore` to restore the data to the new `8.0` instance.
    Here is the command for restoring the data to the new `8.0` instance:
    ```bash
    mongorestore --uri="$MONGODB_URI" --gzip --dir=dump --drop
    ```
    This will restore the data from the dump folder to the new `8.0` instance. The `--drop` flag will drop the database before restoring the data. This is useful if you want to start with a clean database.

### Replica Set Transfer
In this case you will need to transfer the data from the `7.x` instance of the DB to a new `8.0` instance.
1. Create a new `8.0` instance of MongoDB.
2. Add the new instance to the replica set.
3. Use `rs.stepDown()` to step down the `7.x` instance.
4. Use `rs.reconfig()` to reconfigure the replica set to include the new `8.0` instance.
    Here is the command for reconfiguring the replica set:
    ```js
    rs.reconfig({
        "_id": "rs0",
        "version": 2,
        "members": [
            {
                "_id": 0,
                "host": "{MONGO_7_X_PUBLIC_IP}:27017",
            },
            {
                "_id": 1,
                "host": "{MONGO_8_0_PUBLIC_IP}:27017",
            }
        ]
    })
    ```
    This will reconfigure the replica set to include the new `8.0` instance.
5. Use `rs.status()` to check the status of the replica set.

## Conclusion
That was what worked for me. Let me know in the comments if you run into any issues with this process and I'll try to help you out.
