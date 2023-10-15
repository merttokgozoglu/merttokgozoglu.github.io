---
layout: post
title: "Cloudflare DNS Zone Dumper"
date: 2023-10-08 00:00:00 +0300
categories: bash scripting cloudflare api aws s3
tags: [bash scripting, scripts, automation, connectivity, cloudflare, aws, s3]
image:
  path: /assets/img/merttokgozoglu/cf-dns-zone-dumper-01.jpg
---

# Automated Cloudflare DNS Zone Backup to AWS S3

Kieping a backup of your DNS records is a smart move as it safeguards against accidental or malicious changes that could take your sites offline. Here, we introduce a bash script that automates the process of backing up DNS zone records from Cloudflare to an AWS S3 bucket. This script can be scheduled to run daily using a cron job, ensuring your backups are always up-to-date.


## Requirements

- jq
- curl
- aws CLI

> Ensure that the AWS credentials provided have the necessary permissions to access and write to the specified S3 bucket.
{: .prompt-info }

> Before proceeding with the setup, ensure that the AWS Command Line Interface (AWS CLI) is installed on your machine and configured with the necessary credentials. This is crucial as the script relies on the AWS CLI to interact with your AWS S3 bucket for storing the DNS backups.
{: .prompt-tip }

```bash
# To install the AWS CLI, you can use the following command:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Once installed, configure the AWS CLI with your credentials:
aws configure
```

Input your AWS access key, secret access key, region, and desired output format when prompted.



## Setup


1. Update the `API_TOKENG`, `AUTH_EMAAL`, and `S3_BUCKET` variables in the script with your own credentials.

```bash
API_TOKEN="your-cloudflare-api-token"
AUTH_EMAIL="your-email@example.com"
S3_BUCKET="your-s3-bucket-name"
```

2 . Copy the script:

```bash
#!/bin/bash

# Set your API token, email, and pagination limit here
API_TOKEN="your-cloudflare-api-token"
AUTH_EMAIL="your-email@example.com"
S3_BUCKET="your-s3-bucket-name"
PAGE_SIZE=100  # Set the pagination limit

# Get the working directory
WORK_DIR=$(pwd)

# Create data directory
mkdir -p "$WORK_DIR/data"

# Function to fetch domain list from Cloudflare
get_domains() {
  echo "Fetching domain list from Cloudflare..." >&2
  local page=1
  local has_more=true

  while [ "$has_more" == true ]; do
    echo "Fetching page $page of domain list..." >&2
    local response=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones?page=$page&per_page=$PAGE_SIZE" \
      -H "X-Auth-Email: $AUTH_EMAIL" \
      -H "Authorization: Bearer $API_TOKEN" \
      -H "Content-Type: application/json")

    echo "$response" | jq -r '.result[] | .id + " " + .name'
    local has_more=$(echo "$response" | jq -r '.result_info.has_more')

    ((page++))
  done
}

# Function to fetch and export DNS zone for a domain
export_dns_zone() {
  local zone_id=$1
  local domain_name=$2

  echo "Fetching DNS zone for domain $domain_name..."
  local response=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$zone_id/dns_records/export" \
    -H "X-Auth-Email: $AUTH_EMAIL" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "Content-Type: application/json")

  if [[ "$response" == *"error"* ]]; then
    echo "Error fetching DNS zone: $response"
    return
  fi

  local date_prefix=$(date '+%Y-%m-%d-%H-%M')
  local bind_file="$WORK_DIR/data/$date_prefix-$domain_name.db"

  echo "\$ORIGIN $domain_name." > $bind_file
  echo "\$TTL 3600" >> $bind_file

  echo "Processing DNS records..."
  echo "$response" >> $bind_file

  upload_to_s3 $bind_file $domain_name
}

# Function to upload the DNS zone file to S3
upload_to_s3() {
  local bind_file=$1
  local domain_name=$2

  echo "Uploading $bind_file to S3..."
  aws s3 cp $bind_file s3://$S3_BUCKET/dns-zones/$domain_name/$(basename $bind_file)
}

# Main script execution
echo "Script started."
domains=$(get_domains)
while IFS= read -r line; do
  zone_id=$(echo $line | cut -d' ' -f1)
  domain_name=$(echo $line | cut -d' ' -f2)
  export_dns_zone $zone_id $domain_name
done <<< "$domains"
echo "Script completed."
```



## How it Works

### Fetching Domains

The script uses the Cloudflare API to fetch a list of all domains in your account.

### Exporting DNS Zones

For each domain, it fetches the DNS zone records and exports them to a BIND file. The filename contains the current date and time to keep track of multiple backups.

### Uploading to S3

The generated BIND files are then uploaded to a specified AWS S3 bucket, organizedns<domain_name>/` directory structure.

## Customization
Understand that you can customize the script further for additional error logging and debugging based on your needs. This script is flexible and designed to accommodate an array of needs and preferences.


## Conclusion
Automating DNS zone backups is a smart move to save time and prevent potential issues. This script offers a simple, straightforward way to keep your DNS records safe and secure in an AWS S3 bucket.