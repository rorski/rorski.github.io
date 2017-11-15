---
layout: post
title:  "A simple Grafana dashboard S3 backup script"
date:   2017-10-12 08:00:00 -0700
categories: programming aws
---
This is a python script to backup your Grafana dashboards to an S3 bucket. I found [a handful](https://gist.github.com/crisidev/bd52bdcc7f029be2f295) of solutions on Github, but found them a little convoluted, with lots of curl output parsing. This (I think) makes it a little cleaner and more maintanable. So, instead of manually exporting dashboards from the UI, you can just run this on a schedule to backup your dashboards.

This will basically use a token to authenticate to your Grafana host, pull the list of dashboards and write them to a backup directory. Then, boto will backup those json files to the specified S3 directory.

You will of course need boto set up and a bucket you can write to. You will also need a [a token](http://docs.grafana.org/http_api/auth/) to authenticate via HTTP to your server.
{% highlight python %}
import boto
import json
import requests
import sys
import syslog

# This assumes you're doing the backup from the Grafana host itself
HOST = 'http://localhost:3000'
# Create an API token from the admin dashboard and paste it here to enable backups
KEY = ''
# Edit this to write the dashboard json to a location you want
BACKUP_DIR = '/opt/grafana/dashboards/'
# Edit this to use the bucket you want to write backups to
BACKUP_BUCKET = 'my-backup-bucket'

HEADERS = {'Authorization':'Bearer ' + KEY}

def backup_dashboards_to_S3(dashboards):
    """
    Back up dashboards from BACKUP_DIR to S3.
    """
    conn = boto.connect_s3()
    bucket = conn.get_bucket(BACKUP_BUCKET)
    key = boto.s3.key.Key(bucket)

    for dash in dashboards:
        key.name = 'grafana/' + dash + '.json'
        key.set_contents_from_filename(BACKUP_DIR + dash + '.json')

def get_dashboards():
    """
    Returns list of Grafana dashboards
    """
    dashboards = []
    r = requests.get(HOST + '/api/search', headers=HEADERS)

    for dash in r.json():
        dashboards.append(dash['uri'].split('/')[1])

    return dashboards

def main():
    """
    Backup dashboards to BACKUP_DIR
    """
    dashboards = get_dashboards()
    for dash in dashboards:
        dash_json = requests.get(HOST + '/api/dashboards/db/' + dash, headers=HEADERS).json()
        try:
            with open(BACKUP_DIR + dash + '.json', 'w+') as f:
                json.dump(dash_json, f)
        except Exception as err:
            syslog.syslog(syslog.LOG_ERR, sys.argv[0] + ': ' + str(err))
            sys.exit(1)

    backup_dashboards_to_S3(dashboards)

if __name__ == '__main__':
    main()
{% endhighlight %}
