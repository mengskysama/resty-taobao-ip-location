# resty-taobao-ip-location
A simple nginx lua service for ip.taobao.com

# Description

Note that nginx [stream module](https://nginx.org/en/docs/stream/ngx_stream_core_module.html) is required.

Deploy redis and listen in 127.0.0.1 6379 for cache api result.

Tested on Openresty 1.9.15.1.

# Synopsis

```
    server {

        listen              8082;
        listen              [::]:80 ipv6only=on;
        server_name         a.com;

        location /api/ip {

            default_type  text/html;
            resolver      114.114.114.114 8.8.8.8 valid=300s;

            content_by_lua_block {
              local response = nil
              local query_ip = ngx.var.arg_ip

              -- get from cache
              local redis = require "resty.redis"
              local red = redis:new()
              local ok, err = red:connect("127.0.0.1", 6379);
              if not ok then
                ngx.log(ngx.ERR, "[ERROR] Failed to connect redis:", err)
                return ngx.exit(500)
              end
              local res, err = red:get("ip_" .. query_ip)
              if res and res ~= ngx.null then
                response = res
              end

              -- get from taobao ip api
              if response == nil then
                local http = require "resty.http"
                local httpc = http.new()
                httpc:set_timeout(1000)
                local uri = "http://ip.taobao.com/service/getIpInfo.php?ip=" .. query_ip
                local res, err = httpc:request_uri(uri, {
                  method = "GET"
                })
                if res then
                  local json = require("cjson.safe")
                  local j = json.decode(res.body)
                  if j ~= nil then
                    response = res.body
                    -- we cache result if response json
                    red:set("ip_" .. query_ip, response)
                    red:expire("ip_" .. query_ip, 24*3600)
                  end
                else
                  ngx.log(ngx.ERR, "[ERROR] HTTP request api:" .. uri, err)
                end
              end
              red:close()

              -- response
              if response == nil then
                return ngx.exit(500)
              end
              return ngx.say(response)
            }
        }
    }
  ```
  
# Usage

curl http://127.0.0.1:8082/api/ip?ip=101.81.247.204
