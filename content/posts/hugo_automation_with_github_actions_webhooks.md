+++
title = "Automate your Hugo blog with Github Actions and Webhooks"
date = "2023-07-30T17:35:28+05:30"
author = "Dhamith Hewamullage"
cover = "/gears.jpg"
tags = ["automation", "Hugo", "Github", "Github Actions", "Github Webhooks", "Python", "Nginx"]
keywords = ["automation", "Hugo", "Github", "Github Actions", "Github Webhooks", "Python", "Nginx"]
description = "How to automate the Hugo blog deployments using Github Actions and Webhooks."
showFullContent = false
readingTime = true
hideComments = false
+++

When I started this blog, I was using Wordpress. I was familiar with it and maintaining a Mariadb instance in my small server was looking fun for some reason. But naturally my server started getting unwanted requests that were trying to exploit the Wordpress install I was running. Because of that and keeping a Mariadb instance just for a few blog posts beginning to look wasteful, so I moved away from Wordpress to [Hugo](https://gohugo.io/) . Hugo is a very customizable static site generator with support for Markdown formatting, which was simple enough for me to quickly set up. 

With Github Actions and Webhooks, you can update, delete, or publish a new post just by pushing changes to the master branch. All you need is a Github repo to hold your Hugo code, and a server to publish the blog to. 

This is a basic example of how to use Github Webhooks to trigger something externally on a specific event on a repo. Also, the provided Python program can be modified/extended to handle more complex scenarios as required. More information on Webhooks and what are the event types available can be found here: https://docs.github.com/en/webhooks-and-events/webhooks/about-webhooks 

## Step 01: Create a new Github workflow

Once you have set up your Hugo code in a Github repo, you can set up a new Github actions workflow to automate your blog. This is the workflow I am using to build the static site and add it as a new release. There is only one job in the workflow: `build`. It has several steps:

- Install Hugo 
	- Download and install the given Hugo version on the Ubuntu runner
- Build release 
	- Build the static site using Hugo binary
- Pack 
	- Compress the generated static site as a `tar.gz`
- Bump version and push tag 
	- Create a new version and create a new tag
- Upload release
	- Publish the generated static site as a new release

```yml
name: release

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

jobs:
  
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v2
      
      - name: Install Hugo
        run: | 
          wget https://github.com/gohugoio/hugo/releases/download/v0.109.0/hugo_extended_0.109.0_linux-amd64.deb \
          && sudo dpkg -i hugo_extended_0.109.0_linux-amd64.deb 

      - name: Build release
        run: hugo -D 

      - name: Pack
        run: mv public blog && tar -cvf blog.tar.gz blog && sha512sum blog.tar.gz | cut  -f1 -d ' ' | tr -d "\n\r" > hash.txt 

      - name: Bump version and push tag
        id: tag_version
        uses: mathieudutour/github-tag-action@v5.6
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }} 

      - name: Upload Release
        uses: ncipollo/release-action@v1
        with:
          artifacts: 'blog.tar.gz, hash.txt'
          tag: ${{ steps.tag_version.outputs.new_tag }}
          name: Release ${{ steps.tag_version.outputs.new_tag }}
          commit: ${{ steps.vars.outputs.sha }}
          token: ${{ secrets.GITHUB_TOKEN }} 
          makeLatest: true
```


## Step 02: Prepare the server for webhooks

I'm using a small python application to handle the webhook request coming from Github and handle backing up the old blog and deploying the new version. In this program, I only handle the `published` action on release, but you can add/modify this as you wish. This is a simple Python webserver to handle `POST` requests. `validate()` Function validates that request is in fact coming from Github, using the secret key. `handle_release()` Function handles the new release by loading the assets info of the new release and downloading the new build using the `browser_download_url`, then backing up the old version and deploying of the new version.

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
from threading import Thread
import datetime
import hashlib
import hmac
import json
import logging
import os
import requests
import shutil

BLOG_PATH = '/var/www/html/blog'
BACKUP_PATH = '/var/www/html/blog_backup/'
KEY = '' # your secret key shared with Github

class Server(BaseHTTPRequestHandler):
    def do_POST(self):
        res_code = 200
        content_length = int(self.headers['Content-Length'])
        post_data = self.rfile.read(content_length)
        if validate(self.headers['X-Hub-Signature-256'], post_data, KEY.encode('utf-8')):
            try:
                req_body = json.loads(post_data.decode('utf-8'))
                if req_body['action'] == 'published':
                    assets_url = req_body['release']['assets_url']
                    thread = Thread(target=handle_release, args=(assets_url,))
                    thread.start()
            except:
                res_code = 500
        else:
            res_code = 403
        self.send_response(res_code)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write("done".format(self.path).encode('utf-8'))

def handle_release(assets_url):
    r = requests.get(url = assets_url)
    release_file = requests.get(r.json()[0]['browser_download_url'])
    with open('blog.tar.gz', "wb") as f:
        f.write(release_file.content)
    shutil.unpack_archive('blog.tar.gz', '.')
    t = datetime.datetime.now()
    backup = BACKUP_PATH + 'blog_backup_' + t.strftime('%Y%m%d_%H%M%S')
    if os.path.exists('./blog'):
        shutil.move(BLOG_PATH, backup)
        shutil.move('./blog', BLOG_PATH)

def validate(signature, data, key):
    hash_object = hmac.new(key, msg=data, digestmod=hashlib.sha256)
    expected_signature = 'sha256=' + hash_object.hexdigest()
    return hmac.compare_digest(expected_signature, signature)

def main():
    logging.basicConfig(level=logging.INFO)
    server_address = ('', 9999)
    httpd = HTTPServer(server_address, Server)
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        pass
    httpd.server_close()
    logging.info('Stopping server...\n')

if __name__ == '__main__':
    main()
```

Once the Python program is ready, you can just run it on the server, but for better organization, I used Nginx to proxy pass Github webhook requests to the Python application. 

```nginx
server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name 8675.dhamith.me;
        ssl_certificate fullchain.pem;
        ssl_certificate_key privkey.pem;
        access_log /var/log/nginx/webhooks_access.log;
        location / {
            proxy_pass http://127.0.0.1:9999;
        }
}
```

And I have set up the Python program as a service on the server.

```
[Unit]
Description=Github webhook handler
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Type=simple
WorkingDirectory=/home/309/github_webhook
ExecStart=python3 /home/309/github_webhook/main.py

[Install]
WantedBy=multi-user.target
```


## Step 03: Setting up webhooks on Github repo

The next step is to set up webhooks on the Github repo. Go to `Settings -> Webhooks -> Add Webhook` and add a new Webhook. Make sure to use a secure secret when setting up the webhook. And change the `Content type` to `application/json`. I used only `Release` events using the `Let me select individual events` option, but this can be changed as required. Make sure to change the Python program logic to handle the extra events. Once you click on `Add Webhook`, Github will ping the `Payload URL` and if you have correctly set up everything on the server side, the ping should be successful. 

That's it. If everything connects correctly, once you push any changes to the main/master branch, you should see the changes automatically apply on the server. 
