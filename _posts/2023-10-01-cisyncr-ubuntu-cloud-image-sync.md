---
layout: post
title: "CISYNCR: Automating Cloud Images"
date: 2023-10-01 00:00:00 +0300
categories: bash scripting automation cloud-images ubuntu
tags: [bash scripting, scripts, automation, cisyncr, cloud images, ubuntu]
image:
  path: /assets/img/merttokgozoglu/cisyncr-ubuntu-01.jpg
---

CISYNCR is designed to automate the retrieval and management of Ubuntu cloud images for the `amd64` architecture.

## Features
- Automated retrieval of SHA256 checksums.
- Filters for `amd64` architecture.
- Automated download of the latest Ubuntu cloud images.
- Integrity verification using SHA256 checksums.
- Logging and updating a MySQL database.
- Telegram notifications for updates.

## Prerequisites
- Bash shell
- curl
- MySQL client
- Access to a Telegram bot for notifications

## Setup and Usage
1. Get the [`cisyncr-ubuntu.sh`](#cisyncr-ubuntush) and [`database.sql`](#databasesql) file.

2. Make the script executable and import database.
    ```bash
    chmod +x ubuntu-cisyncr.sh
    mysql < database.sql
    ```
3. Update the configuration section at the beginning of the script with your specific details.
4. Execute the script:
    ```bash
    ./cisyncr-ubuntu.sh
    ```

## Files

### cisyncr-ubuntu.sh
```bash
#!/bin/bash

# Configuration variables
TELEGRAM_BOT_TOKEN="0987654321:TOKEN"
TELEGRAM_CHAT_ID="-123456789"
MYSQL_USER="user"
MYSQL_PASS="pass"
MYSQL_HOST="host"
MYSQL_DB="ubuntu_cloud_images"
BASE_URL="https://cloud-images.ubuntu.com"
DESTINATION_PATH="images"
mkdir $DESTINATION_PATH

# Function to send Telegram alert
send_telegram() {
    local message=$1
    curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" -d chat_id=$TELEGRAM_CHAT_ID -d text="$message" > /dev/null
}

# Function to check and download Ubuntu images
check_and_download_ubuntu() {
    local version=$1

    # Fetch SHA256SUMS file
    curl -s -O $BASE_URL/$version/current/SHA256SUMS

    # Filter out unwanted architectures
    sed -i '/ppc64el\|arm64\|i386\|s390x\|armhf\|powerpc\|uefi/d' SHA256SUMS

    echo "Filtered contents of SHA256SUMS for $version:"
    cat SHA256SUMS
    echo "--------------------------"

    # Extract line with .img that contains both "amd64" and "cloudimg" from SHA256SUMS
    IMG_SUM_LINE=$(grep "\.img" SHA256SUMS | grep "amd64" | grep "cloudimg")

    # If no matching .img found, then check for just "amd64" and ".img"
    if [[ -z "$IMG_SUM_LINE" ]]; then
        IMG_SUM_LINE=$(grep "\.img" SHA256SUMS | grep "amd64")
    fi

    # If still no matching .img found, log an error and exit this function iteration
    if [[ -z "$IMG_SUM_LINE" ]]; then
        echo "No suitable .img found for $version. Skipping..."
        send_telegram "No suitable .img found for $version. Please check the source!"
        return
    fi

    echo "Extracted suitable .img line: $IMG_SUM_LINE"

    # Split checksum and filename
    CHECKSUM=$(echo $IMG_SUM_LINE | awk '{print $1}')
    IMG_FILE=$(echo $IMG_SUM_LINE | awk '{print $2}'|sed 's/*//g')

    # Check if image exists
    if [[ -f "$DESTINATION_PATH/$IMG_FILE" ]]; then
        LOCAL_CHECKSUM=$(sha256sum "$DESTINATION_PATH/$IMG_FILE" | awk '{print $1}')
        if [[ "$LOCAL_CHECKSUM" == "$CHECKSUM" ]]; then
            echo "File $IMG_FILE is up-to-date."
            # Update the last check timestamp for this image
            mysql -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASS $MYSQL_DB -e "UPDATE images SET last_check_timestamp=NOW() WHERE filename='$IMG_FILE';"
            return
        else
            # If there's a new version, mark the old one as not current
            mysql -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASS $MYSQL_DB -e "UPDATE images SET current_version=false WHERE filename='$IMG_FILE';"
        fi
    else
        echo "File $IMG_FILE not found locally. Downloading..."
    fi

    # Download .img file
    curl -s -O $BASE_URL/$version/current/$IMG_FILE

    # Verify the checksum
    echo "$CHECKSUM  $IMG_FILE" | sha256sum --check -

    if [[ $? -eq 0 ]]; then
        # Move file if integrity is confirmed
        mv $IMG_FILE $DESTINATION_PATH/

        # Log to database
        mysql -h $MYSQL_HOST -u $MYSQL_USER -p$MYSQL_PASS $MYSQL_DB -e "INSERT INTO images (operating_system, distribution, distro_version, filename, current_version, genesis_timestamp) VALUES ('Linux', 'Ubuntu', '$version', '$IMG_FILE', true, NOW()) ON DUPLICATE KEY UPDATE current_version=true, last_check_timestamp=NOW();"

        # Send Telegram alert
        send_telegram "File $IMG_FILE has been downloaded, verified, and moved successfully."
    else
        # Notify about the mismatch
        send_telegram "Checksum verification failed for $IMG_FILE. The file has not been moved."
    fi
}

# Check and download for specified Ubuntu versions
for version in "focal" "bionic" "xenial" "trusty"; do
    check_and_download_ubuntu $version
done
```

Set up your MySQL database using the following dump:
### database.sql
```sql
-- MySQL dump 10.19  Distrib 10.3.38-MariaDB, for debian-linux-gnu (x86_64)
--
-- Host: localhost    Database: ubuntu_cisyncr
-- ------------------------------------------------------
-- Server version       10.3.38-MariaDB-0ubuntu0.20.04.1

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `images`
--

DROP TABLE IF EXISTS `images`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `images` (
  `image_id` int(11) NOT NULL AUTO_INCREMENT,
  `operating_system` varchar(255) NOT NULL,
  `distribution` varchar(255) NOT NULL,
  `distro_version` varchar(255) NOT NULL,
  `filename` varchar(255) NOT NULL,
  `current_version` tinyint(1) DEFAULT NULL,
  `genesis_timestamp` datetime NOT NULL,
  `last_check_timestamp` datetime DEFAULT NULL,
  PRIMARY KEY (`image_id`)
) ENGINE=InnoDB AUTO_INCREMENT=39 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `images`
--

LOCK TABLES `images` WRITE;
/*!40000 ALTER TABLE `images` DISABLE KEYS */;
INSERT INTO `images` VALUES (14,'Linux','Ubuntu','focal','focal-server-cloudimg-amd64.img',0,'2023-09-10 23:58:18','2023-09-11 00:07:13'),(15,'Linux','Ubuntu','bionic','bionic-server-cloudimg-amd64.img',0,'2023-09-10 23:58:30','2023-09-26 18:04:03'),(16,'Linux','Ubuntu','xenial','xenial-server-cloudimg-amd64-disk1.img',0,'2023-09-10 23:58:41','2023-09-26 18:04:06'),(17,'Linux','Ubuntu','trusty','trusty-server-cloudimg-amd64-disk1.img',0,'2023-09-10 23:58:50','2023-09-11 00:07:21'),(18,'Linux','Ubuntu','focal','focal-server-cloudimg-amd64.img',0,'2023-09-10 23:59:46','2023-09-11 00:07:13'),(19,'Linux','Ubuntu','bionic','bionic-server-cloudimg-amd64.img',0,'2023-09-10 23:59:59','2023-09-26 18:04:03'),(20,'Linux','Ubuntu','xenial','xenial-server-cloudimg-amd64-disk1.img',0,'2023-09-11 00:00:57','2023-09-26 18:04:06'),(21,'Linux','Ubuntu','trusty','trusty-server-cloudimg-amd64-disk1.img',0,'2023-09-11 00:01:06','2023-09-11 00:07:21'),(22,'Linux','Ubuntu','focal','focal-server-cloudimg-amd64.img',0,'2023-09-11 00:02:36','2023-09-11 00:07:13'),(23,'Linux','Ubuntu','bionic','bionic-server-cloudimg-amd64.img',1,'2023-09-11 00:02:49','2023-09-26 18:04:03'),(24,'Linux','Ubuntu','xenial','xenial-server-cloudimg-amd64-disk1.img',1,'2023-09-11 00:03:01','2023-09-26 18:04:06'),(25,'Linux','Ubuntu','trusty','trusty-server-cloudimg-amd64-disk1.img',1,'2023-09-11 00:03:10','2023-09-11 00:07:21'),(26,'Linux','CentOS','8','CentOS-8-ec2-8.4.2105-20210603.0.x86_64.qcow2\">CentOS-8-ec2-8.4.2105-20210603.0.x86_64.qcow2',1,'2023-09-11 00:19:46','2023-09-11 00:19:54'),(27,'Linux','CentOS','9-stream','CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2',0,'2023-09-11 01:16:41','2023-09-11 02:14:26'),(28,'Linux','CentOS','9-stream','CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2',0,'2023-09-11 01:22:20','2023-09-11 02:14:26'),(29,'Linux','CentOS','9-stream','CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2',0,'2023-09-11 01:31:41','2023-09-11 02:14:26'),(30,'Linux','CentOS','9-stream','CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2\nCentOS-Stream-GenericCloud-9-latest.x86_64.qcow2',1,'2023-09-11 04:35:05',NULL),(31,'Linux','CentOS','8-stream','CentOS-Stream-GenericCloud-8-latest.x86_64.qcow2\nCentOS-Stream-GenericCloud-8-latest.x86_64.qcow2',1,'2023-09-11 04:35:55',NULL),(32,'Linux','CentOS','8','CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2',1,'2023-09-11 04:36:34',NULL),(33,'Linux','CentOS','9-stream','CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2\nCentOS-Stream-GenericCloud-9-latest.x86_64.qcow2',1,'2023-09-11 04:39:00',NULL),(34,'Linux','CentOS','9-stream','CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2',0,'2023-09-11 04:40:41',NULL),(35,'Linux','CentOS','9-stream','CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2',1,'2023-09-11 04:43:27',NULL),(36,'Linux','CentOS','9-stream','CentOS-Stream-GenericCloud-9-20230904.0.x86_64.qcow2',0,'2023-09-11 05:01:21',NULL),(37,'Linux','Ubuntu','focal','focal-server-cloudimg-amd64.img',1,'2023-09-26 18:03:58',NULL),(38,'Linux','Ubuntu','trusty','trusty-server-cloudimg-amd64-disk1.img',1,'2023-09-26 18:04:15',NULL);
/*!40000 ALTER TABLE `images` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2023-09-26 18:14:55
```