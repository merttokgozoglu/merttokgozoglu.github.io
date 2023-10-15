---
layout: post
title: "Bashtion: Your Smart SSH Bastion Connector"
date: 2023-10-15 00:00:00 +0300
categories: bash scripting ssh bastion-host
tags: [bash scripting, scripts, automation, bashtion, bastion host, connectivity]

image:
  path: /assets/img/merttokgozoglu/bashtion-01.jpg
---

Bashtion is a sleek, user-friendly SSH bastion host connector scripted in Bash, making server connections straightforward and swift. By melding the robust SSH protocol with an intuitive CLI menu, Bashtion stands as a reliable bridge to your servers.

## Features

- **Static File Generation**: Create a static file from a host database to speed up access and connectivity.
- **Keyword-based Search**: Find and connect to a host using keywords or specific IDs.
- **Interactive Menu**: A user-centric menu to list and select the servers you want to connect to.
- **Automatic SSH**: Auto-initiate an SSH connection once a server is picked.

## How to Use

1. **Generate Static File**: 
    ```bash
    ./bashtion.sh generate
    ```

2. **Connect to a Server**:
    - Via ID or Keywords:
        ```bash
        ./bashtion.sh [ID/Keywords]
        ```
    - Via Interactive Menu:
        ```bash
        ./bashtion.sh
        ```

## Database Structure

Ensure your database file (`host.db`) follows this structure:
```plaintext
Hostname        IP Address
FRA1-GRAGLOY    172.16.101.33
IST2-WAZUH      172.16.102.11
LON9-GRAFANA    172.16.109.22
```

## Sample Output

```plaintext
$ ./bashtion.sh
==============================
|       SSH BASTION HOST      |
==============================

1   IST1-N01                  | 24  IST1-Client-DB3           |
2   IST1-N02                  | 25  IST1-Client-Memcache      |
3   IST1-N03                  | 26  IST1-Client-Job           |
4   IST1-N04                  | 27  IST1-Client-Queue         |
5   IST1-N05                  | 28  IST1-Client-Git           |
6   IST1-N06                  | 29  IST1-Client-Dev           |
7   IST1-N07                  | 30  IST2-Client-pfSense       |
8   IST1-N08                  | 31  IST2-Client-LB            |
9   IST1-N09                  | 32  IST2-Client-APP1          |
10  IST1-Wazuh                | 33  IST2-Client-APP2          |
11  IST1-pfSense              | 34  IST2-Client-APP3          |
12  IST1-Netbox               | 35  IST2-Client-NFS           |
13  IST1-Linda                | 36  IST2-Client-DB            |
14  IST1-Proxy                | 37  IST2-Client-Redis         |
15  IST1-Ubuntu               | 38  ANK1-N01                  |
16  IST1-DEV                  | 39  ANK1-N02                  |
17  IST1-PBS                  | 40  SYS3-TOKGOZOGLU-NET       |
18  IST1-PMG                  | 41  ANK1-pfSense              |
19  IST1-Zimbra               | 42  IST2-N01                  |
20  IST1-DB1                  | 43  IST2-N02                  |
21  IST1-DB2                  | 44  IST2-N03                  |
22  IST1-Jitsi                | 45  IST2-N04                  |
23  IST1-Desktop              | 46  IST2-N05                  |
```

## bashtion.sh

```bash
#!/bin/bash

DB_FILE="host.db"
STATIC_FILE="static_output.txt"

generate_static_file() {
    if [ ! -f "$DB_FILE" ]; then
        echo "Error: $DB_FILE not found!"
        exit 1
    fi

    > "$STATIC_FILE"

    index=1
    col_count=0
    declare -a columns

    while read -r line; do
        name=$(echo "$line" | awk '{print $1}')
        ip=$(echo "$line" | awk '{print $2}')

        name=${name:0:25}

        columns[$col_count]+=$(printf "%-3s %-25s | " "$index" "$name")
        echo "$index:$ip:$name" >> "$STATIC_FILE"

        ((index++))
        ((col_count++))
        [ "$col_count" -ge 30 ] && col_count=0

    done < "$DB_FILE"

    for col in "${columns[@]}"; do
        [ -n "$col" ] && echo "$col" >> "$STATIC_FILE"
    done
}

connect_by_keywords() {
    keywords="$1"
    IFS=',' read -ra ADDR <<< "$keywords"
    
#    results="$DB_FILE"
    results=$(cat "$DB_FILE")
    for i in "${ADDR[@]}"; do
        results=$(echo "$results" | grep -i "$i")
    done

    count=$(echo "$results" | wc -l)

    if [ "$count" -eq 1 ]; then
        ip=$(echo "$results" | awk '{print $2}')
        ssh root@"$ip"
    else
        echo "$count servers found. Please be more specific or use ID to connect directly."
        echo "$results"
    fi
}

display_menu() {
    echo "=============================="
    echo "|       SSH BASTION HOST      |"
    echo "=============================="
    echo ""

    if [ ! -f "$STATIC_FILE" ]; then
        echo "Error: Static file missing. Please run './bashtion.sh generate' first!"
        exit 1
    fi

    awk -F":" 'NF == 1' "$STATIC_FILE"
    echo ""
    read -p "Select server to connect to (Enter ID or keywords): " choice

    if [[ $choice =~ ^[0-9]+$ ]]; then
        connect_to_server "$choice"
    else
        connect_by_keywords "$choice"
    fi
}

connect_to_server() {
    choice=$1
    ip=$(awk -F":" -v id="$choice" '$1 == id { print $2 }' "$STATIC_FILE")

    if [ -z "$ip" ]; then
        echo "Invalid choice. Exiting."
        exit 1
    else
        ssh root@"$ip"
    fi
}

case $1 in
    generate)
        generate_static_file
        ;;
    *)
        if [[ -z $1 ]]; then
            display_menu
        elif [[ $1 =~ ^[0-9]+$ ]]; then
            connect_to_server "$1"
        else
            connect_by_keywords "$1"
        fi
        ;;
esac

exit 0

```