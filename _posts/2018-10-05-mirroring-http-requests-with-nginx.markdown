---
layout: post
title:  "Mirroring HTTP requests with Nginx"
date:   2018-10-05 13:58 +0300
categories: Nginx
---

# Mirroring HTTP requests with Nginx

I recently needed to mirror certain HTTP traffic going through an Nginx reverse proxy servers, to a 
special endpoint, without (of course) interrupting any actual traffic.

##### Several things that came to mind right away were:

1. What happens with responses (if any) to mirrored requests?
2. How can this be achieved?
3. Can this change somehow have any impact on the reliability of the service in terms of uptime / performance?

The basic flow as it was before:

```
server  {

  listen  443 ssl;  
  server_name  x.x.net;
  ssl  on;
  client_max_body_size 0;
  location / {
             proxy_pass https://b.b;
             proxy_pass_request_headers on;
             proxy_pass_request_body on;
             
            }
            
  }
```

This will ensure any requests reaching our proxy and that were directed towards "https://x.x.net", will be passed over
to the destination and the response back to the client, poorly illustrated below:

![simple flow](/assets/simple_flow_1.jpg)

##### Now, let's try to address the thoughts above:

1. We partially touche #3, but -  Nginx (Version `1.13.4` and above) has a built in module called [http mirror module](http://nginx.org/en/docs/http/ngx_http_mirror_module.html).
    
    This module implements mirroring of any original request by creating a subrequest - and any responses to the subrequest 
    are dropped - which is exactly what I needed.

2.  Other than the obvious Nginx version upgrade as needed (mentioned above), we will need to change the configuration of Nginx slightly to support this.
    
    In our existing location, we need to add the name of the new mirror (will be created shortly), and whether or not we choose to mirror the POST body as well, or not.
    
    ```
      location / {
      
             mirror /root_mirror;
             mirror_request_body on;
             proxy_pass https://b.b;
             proxy_pass_request_headers on;
             proxy_pass_request_body on;
             
    }
    ```
    
    Now, we will need to create a new location, for the /root_mirror (The one we reference above):
    
    ```
      location = /root_mirror {
            proxy_pass https://special.endpoint.com;
            proxy_pass_request_body on;
    }
    ```
    
    Final configuration should look something like:
    
    ```
    server  {
    
    listen  443 ssl;  
    server_name  x.x.net;
    ssl  on;
    client_max_body_size 0;
    location / {
    
             mirror /root_mirror;
             mirror_request_body on;
             proxy_pass https://b.b;
             proxy_pass_request_headers on;
             proxy_pass_request_body on;
             
            }
            
    location = /root_mirror {
    proxy_pass https://special.endpoint.com;
    proxy_pass_request_body on;
    
        }
    }
    ```
    
    ![Modified Flow](/assets/full_flow_2.jpg)
    
    Now, will it work? yes. Should you apply this it on production? See answer #3 below.
    
3. So, there are several implications to the above configuration:

    - You just made your production less reliable:
    
        With this configuration, Nginx will use the default libc resolver to attempt and resolve to your mirrored endpoint.   
        If that ever becomes unavailable, so does your nginx. the service will even stop and refuse to start: `upstream host is unavailable`.
        
        A workaround for this would be to use a dedicated 'local' resolver for this upstream, and store it in a variable, as below:
        
        ```
          location = /root_mirror {
            resolver 10.0.23.5; # local resolver IP example
            set $upstream_endpoint https://special.endpoint.com; # only this endpoint will be resolved via the local resolver
            proxy_pass $upstream_endpoint;
            proxy_pass_request_body on;
        }
        ```
     - Even with the above, your prod is still less reliable - slow resolving or processing from the "mirror" side will impact the overall time
     it takes for Nginx to pass the response back to the original request.
     
     Consider adding timeouts in the mirrored location, so let's say if after 0.2 second (Don't mind the value now...) it did not respond, Nginx will drop that mirrored request but
     continue serving the original request. `proxy_connect_timeout 200ms;` `proxy_read_timeout 200ms;`.
     
     