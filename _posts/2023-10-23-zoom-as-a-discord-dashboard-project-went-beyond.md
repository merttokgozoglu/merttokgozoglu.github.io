---
layout: post
title: "The Lookout Project: Zoom as a Discord"
date: 2023-10-23 10:20:00 +0300
categories: data visualization integration
tags: [data, csv, visualization, zoom, discord, webhooks, nginx, mongodb, python, php]
image:
  path: /assets/img/merttokgozoglu/lookout-01.jpg
---


The journey of Lookout began with a modest aim: to enhance the management of a Zoom Personal Meeting Room for a tightly-knit community. The idea was to emulate the Discord server experience, especially during screen sharing and other interactive sessions, all without necessitating Pro subscriptions for every participant. As the endeavor progressed, the Zoom room not only mirrored the interaction facilitation of a Discord server but also started keeping a vigilant eye on the entrances and exits of participants, akin to a friendly neighborhood watch. This dual capability of fostering seamless interactions and monitoring participant activity birthed the project name "Lookout". It's like having a cozy little Discord server, but within Zoom, keeping tabs on the comings and goings, ensuring the room remains a lively and interactive space for trusted cohorts. This evolution, perhaps sprinkled with a dash of coding mischief, embodies the spirit of Lookout, balancing functionality with a touch of vigilance.


## Example Live Dashboard
{% assign data_file = site.data['zoom-room-dashboard'] %}

<table>
  {% for row in data_file %}
    {% if forloop.first %}
      <tr>
        {% for pair in row %}
          <th>{{ pair[0] }}</th>
        {% endfor %}
      </tr>
    {% endif %}
  
    {% tablerow pair in row %}
      {{ pair[1] }}
    {% endtablerow %}
  {% endfor %}
</table>
> In light of maintaining privacy and respecting the digital boundaries of all participants, the Lookout dashboard has been designed to exclusively track and display my online status within the Zoom room, without revealing the online status or any other details of other participants. The dashboard dynamically updates to reflect my real-time status, ensuring that while the essence of monitoring is preserved, it’s done without encroaching on the privacy of any individuals involved. This selective visibility ensures that the room remains a secure and comfortable environment for everyone, while still embodying the essence of a close-knit community akin to a personal Discord server.
{: .prompt-info }



## Prerequisites

Before diving into the setup, ensure the following prerequisites are met:

<table style="border-collapse: collapse; border: 0;">
    <tr>
        <td><img src="/assets/img/merttokgozoglu/zoom-logo.png" alt="Zoom" style="max-height:100px; width:auto;"></td>
        <td><img src="/assets/img/merttokgozoglu/mongodb-logo.png" alt="MongoDB" style="max-height:100px; width:auto;"></td>
        <td><img src="/assets/img/merttokgozoglu/python-logo.png" alt="Python" style="max-height:100px; width:auto;"></td>
        <td><img src="/assets/img/merttokgozoglu/php-logo.jpg" alt="PHP" style="max-height:100px; width:auto;"></td>
        <td><img src="/assets/img/merttokgozoglu/mattermost-logo.webp" alt="Mattermost" style="max-height:100px; width:auto;"></td>
        <td><img src="/assets/img/merttokgozoglu/nginx-logo.jpg" alt="Nginx" style="max-height:100px; width:auto;"></td>
        <td><img src="/assets/img/merttokgozoglu/github-logo.png" alt="GitHub" style="max-height:100px; width:auto;"></td>
    </tr>
</table>


### 1. **Zoom Account and Webhook-only App**:
   - A Zoom Pro account to create a Personal Meeting Room.
   - Create a [Webhook-only App](https://marketplace.zoom.us/docs/guides/build/webhook-only-app) on the [Zoom App Marketplace](https://marketplace.zoom.us/) for real-time event notifications.

### 2. **Server Setup**:
   - A server with a public IP for hosting the webhook endpoint, running MongoDB, and serving the `_data` folder via Nginx.
   - Install PHP and the required extensions for running the Zoom webhook scripts.
   - Install Python for running the data processing script.

> **Webhook Verification:**
> Before proceeding with the setup, it's crucial to first verify the webhook. Upon verification, make sure to switch the Nginx configuration to handle the webhook endpoint accordingly. This step is essential to ensure the secure and correct functioning of the webhook integration with your Zoom room.
{: .prompt-warning }

### 3. **Database**:
   - Install [MongoDB](https://docs.mongodb.com/manual/installation/) on the server.
   - Optionally, install [Mongo Compass](https://www.mongodb.com/products/compass) for easier query crafting and data visualization.

### 4. **Messaging Platform**:
   - Set up a [Mattermost server](https://docs.mattermost.com/guides/administrator.html#installing-mattermost) for notifications from the Zoom webhook scripts.

### 5. **Web Server**:
   - Install and configure [Nginx](https://nginx.org/en/docs/) as per the provided configuration to serve the `_data` folder and proxy requests to GitHub Pages.

### 6. **GitHub Repository**:
   - A GitHub repository to host the Jekyll-based website for data visualization.
   - A basic understanding of [Jekyll](https://jekyllrb.com/docs/) and [GitHub Pages](https://pages.github.com/) is useful.

## Technical Architecture

Lookout consists of several components that work together to collect, store, process, and visualize data. Here's a detailed breakdown of the system architecture:

### 1. Data Collection

#### Zoom Webhook Integration:

Utilize Zoom's Webhook-only app to receive real-time notifications of participant actions in the Zoom room. Configure the webhooks to send POST requests to a designated endpoint on your server, which is processed by a PHP script.

##### Webhook Validation and Processing Scripts:

- `zoom_validate.php`: Validates the webhook endpoint according to Zoom's requirements initially.
- `zoom_hooks.php`: Replaces `zoom_validate.php` post-validation to handle incoming webhook notifications, extracts the data from the POST request, processes it, then stores it in the MongoDB database, and sends a notification to a Mattermost channel.

### 2. Data Storage

#### MongoDB:

MongoDB is chosen for its powerful querying capabilities and speed in delivering query results, making it an excellent choice for handling real-time data.

### 3. Data Processing

#### Python Script (`zoom-dashboard.py`):

This script fetches data from MongoDB, processes it to identify the current and past status of each participant, and then generates a `zoom-room-dashboard.csv` file with this information.

### 4. Data Visualization

#### Jekyll and GitHub Pages:

Place the CSV file in the `_data` folder of a GitHub repository hosting a Jekyll-based website. A markdown file under the `posts` folder renders and displays the data from the CSV file in a tabular format on a webpage.

### 5. Web Server Configuration

#### Nginx:

Configure an Nginx server to serve the `_data` folder from a different domain, bypassing the need to constantly push updates to GitHub. This setup also resolves cross-origin issues, ensuring seamless data update and retrieval.

### 6. Automation

#### Crontab:

Employ Crontab to ensure the Python script runs at regular intervals, keeping the data up-to-date without manual intervention.

## Code Snippets and Configuration Details

Here are the crucial snippets and configurations used in Lookout:

### Zoom Webhook PHP Scripts:

```php
// zoom_validate.php

<?php
// zoom_validate.php

// Your MongoDB configuration
$mongo_user = 'dbuser';
$MONGO_PASSWORD = 'dbpassword';
$MONGO_HOST = '127.0.0.1';
$MONGO_PORT = '27017';
$MONGO_DB = 'zoomEvents';

// Create MongoDB connection
$manager = new MongoDB\Driver\Manager("mongodb://$MONGO_USER:$MONGO_PASSWORD@$MONGO_HOST:$MONGO_PORT");

// Your Mattermost webhook URL
$MATTERMOST_WEBHOOK_URL = 'https://mattermost.domain.tld/hooks/hook-random-id';

// Get the Zoom webhook request body
$requestBody = file_get_contents('php://input');
$data = json_decode($requestBody, true);

// Store data in MongoDB
$bulk = new MongoDB\Driver\BulkWrite;
$bulk->insert($data);  // Changed this line
$manager->executeBulkWrite("$MONGO_DB.zoomEvents", $bulk);

// Construct the message to send to Mattermost
$message = "Zoom Event: " . $data['event'] . "\n" . json_encode($data['payload'], JSON_PRETTY_PRINT);

// Prepare the payload for Mattermost
$payload = json_encode(['text' => $message]);

// Send the message to Mattermost
$ch = curl_init($MATTERMOST_WEBHOOK_URL);
curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "POST");
curl_setopt($ch, CURLOPT_POSTFIELDS, $payload);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    'Content-Type: application/json',
    'Content-Length: ' . strlen($payload)
]);

$result = curl_exec($ch);
curl_close($ch);

?>
```

```php
// zoom_hooks.php

<?php
// zoom_hooks.php

// Your MongoDB configuration
$mongo_user = 'dbuser';
$MONGO_PASSWORD = 'dbpassword';
$MONGO_HOST = '127.0.0.1';
$MONGO_PORT = '27017';
$MONGO_DB = 'zoomEvents';
$MONGO_COLLECTION = 'events';

// Create MongoDB connection
$manager = new MongoDB\Driver\Manager("mongodb://$MONGO_USER:$MONGO_PASSWORD@$MONGO_HOST:$MONGO_PORT");

// Your Mattermost webhook URL
$MATTERMOST_WEBHOOK_URL = 'https://mattermost.domain.tld/hooks/hook-random-id';

// Get the Zoom webhook request body
$requestBody = file_get_contents('php://input');
$data = json_decode($requestBody, true);

// Store data in MongoDB
$bulk = new MongoDB\Driver\BulkWrite;
$bulk->insert($data);
$manager->executeBulkWrite("$MONGO_DB.$MONGO_COLLECTION", $bulk);

// Construct the message to send to Mattermost
$message = "Zoom Event: " . $data['event'] . "\n" . json_encode($data['payload'], JSON_PRETTY_PRINT);

// Prepare the payload for Mattermost
$payload = json_encode(['text' => $message]);

// Send the message to Mattermost
$ch = curl_init($MATTERMOST_WEBHOOK_URL);
curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "POST");
curl_setopt($ch, CURLOPT_POSTFIELDS, $payload);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    'Content-Type: application/json',
    'Content-Length: ' . strlen($payload)
]);

$result = curl_exec($ch);
curl_close($ch);

?>
```

### Python Script:

```python
# zoom-dashboard.py
import pymongo
from datetime import datetime
import pandas as pd
import pytz

# Set the timezone
tz = pytz.timezone('Europe/Istanbul')

# MongoDB connection details
mongo_user = 'dbuser'
mongo_password = 'dbpassword'
mongo_host = '127.0.0.1'
mongo_port = '27017'
mongo_db = 'zoomEvents'
mongo_collection = 'events'

# Connect to MongoDB
client = pymongo.MongoClient(f"mongodb://{mongo_user}:{mongo_password}@{mongo_host}:{mongo_port}")
db = client[mongo_db]
collection = db[mongo_collection]

# Prepare a DataFrame for the output
df_output = pd.DataFrame(columns=['Username', 'Status', 'Last Join', 'Last Left'])

# Get unique usernames from the entire collection
pipeline = [
    {"$group": {
        "_id": None,
        "user_names": {"$addToSet": "$payload.object.participant.user_name"}
    }}
]
unique_usernames = list(collection.aggregate(pipeline))[0]['user_names']

# Loop through each unique username and get their actions
for username in unique_usernames:
    user_events = list(collection.find(
        {"payload.object.participant.user_name": username},
        {"event": 1, "event_ts": 1}
    ).sort("event_ts", pymongo.DESCENDING))

    if user_events:
        last_join_timestamp = "N/A"
        last_left_timestamp = "N/A"
        for user_event in user_events:
            event_type = user_event['event']
            event_timestamp_utc = datetime.utcfromtimestamp(user_event['event_ts'] / 1000)
            event_timestamp_istanbul = event_timestamp_utc.astimezone(tz).strftime('%Y-%m-%d-%H:%M:%S')
            if event_type == 'meeting.participant_joined' and last_join_timestamp == "N/A":
                last_join_timestamp = event_timestamp_istanbul
            elif event_type == 'meeting.participant_left' and last_left_timestamp == "N/A":
                last_left_timestamp = event_timestamp_istanbul

            if last_join_timestamp != "N/A" and last_left_timestamp != "N/A":
                break

        status_icon = '<i class=\"fa-solid fa-toggle-on\"></i>' if last_join_timestamp != "N/A" and (last_left_timestamp == "N/A" or last_join_timestamp > last_left_timestamp) else '<i class=\"fa-solid fa-toggle-off\"></i>'
        df_output.loc[len(df_output)] = [username, status_icon, last_join_timestamp, last_left_timestamp]

df_output['Last Action'] = df_output[['Last Join', 'Last Left']].max(axis=1)
df_output = df_output.sort_values(by='Last Action', ascending=False).drop(columns=['Last Action'])

# Save the output as a CSV file
csv_file_path = 'zoom-room-dashboard.csv'
df_output.to_csv(csv_file_path, index=False, quoting=1)
```

### Nginx Configuration:

```plaintext
# /etc/nginx/conf.d/mert.tokgozogu.net.conf
erver {
    listen 80;
    server_name mert.tokgozoglu.net;

    # Redirect all HTTP requests to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name mert.tokgozoglu.net;

    # SSL parameters
    ssl_certificate     /etc/nginx/ssl/CF-ORIGIN.crt;
    ssl_certificate_key /etc/nginx/ssl/CF-ORIGIN.key;

    # Access and Error logs with specified format
    access_log /var/log/nginx/mert.tokgozoglu.net-access.log upstreamli;
    error_log /var/log/nginx/mert.tokgozoglu.net-error.log;

    # Proxy settings for _data folder
    location ^~ /_data {
        alias /var/www/vhosts/mert.tokgozoglu.net/_data;
        expires -1;
        index index.html;
        add_header Cache-Control "no-store, no-cache, must-revalidate, proxy-revalidate";
    }

    # Proxy settings for all other requests
    location / {
        proxy_pass http://merttokgozoglu.github.io;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Forwarded $proxy_add_forwarded;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
    }
}
```

```markdown
### Jekyll Markdown File for Visualization:

```markdown
---
layout: post
title: "Displaying CSV Data"
date: 2023-10-23 00:00:00 +0300
categories: data visualization csv
tags: [data, csv, visualization]
---
## Zoom Room Dashboard {% raw %}
<table>
  {% for row in data_file %}
    {% if forloop.first %}
      <tr>
        {% for pair in row %}
          <th>{{ pair[0] }}</th>
        {% endfor %}
      </tr>
    {% endif %}
  
    {% tablerow pair in row %}
      {{ pair[1] }}
    {% endtablerow %}
  {% endfor %}
</table>{% endraw %}
```

## Future Improvements

The current setup is focused on gathering the essential data and displaying it on the dashboard. However, the Mattermost notifications generated by the `zoom_hooks.php` script are quite basic and could use some refinements for better readability and aesthetics. This refinement can include formatting the messages to make them visually appealing and informative, which will be looked into in future iterations of the project.

## Conclusion

Lookout successfully transforms a Zoom Personal Meeting Room into a vigilant watchtower, keeping tabs on participant activity through a comprehensive dashboard. The blend of webhooks, server configurations, data processing, and visualization makes the Zoom room a lively and interactive space, much akin to a personal Discord server.

