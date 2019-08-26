监控原理：
参考zabbix监控http 5xx，nginx开启stub_status（stub_status on）

1.nginx开启stub_status（stub_status on）
cat /usr/local/nginx/conf/conf.d/status/nginx_status.conf

server {

    listen 60913;

    server_name  127.0.0.1;

    access_log  logs/status/access_status.log ppformat;

    error_log   logs/status/error_status.log;

 

    location /nginx_status {

        stub_status on;

        allow 192.168.0.0/16;

        allow 10.0.0.0/8;

        allow 127.0.0.1;

        deny all;

    }

 

    location /rc {

        req_status_show general;

        allow 192.168.0.0/16;

        allow 10.0.0.0/8;

        allow 127.0.0.1;

        deny all;

    }

}

---------------------------------------------------------------
root@local:~# cat /etc/zabbix/scripts/discovery/domain_discovery.sh
#!/bin/bash

domains=$(wget  -q 127.0.0.1:60913/rc -O - 2> /dev/null | awk -F',' '{print $1"/"$2}')

echo "{"
echo "\"data\":["

comma=""
for domain in $domains
do
    echo "    $comma{\"{#DOMAINNAME_IP}\":\"$domain\"}"
    comma=","
done

echo "]"
echo "}"

root@wlocal:~#
 
root@local:~# cat /etc/zabbix/scripts/nginx/nginx.sh
#!/bin/bash

# Zabbix requested parameter
ZBX_REQ_DATA="$1"
ZBX_REQ_DATA_URL="$2"

# Nginx defaults
NGINX_STATUS_DEFAULT_URL="http://127.0.0.1:60913/nginx_status"
WGET_BIN="/usr/bin/wget"

#
# Error handling:
# - need to be displayable in Zabbix (avoid NOT_SUPPORTED)
# - items need to be of type "float" (allow negative + float)
#
ERROR_NO_ACCESS_FILE="-0.9900"
ERROR_NO_ACCESS="-0.9901"
ERROR_WRONG_PARAM="-0.9902"
ERROR_DATA="-0.9903" # either can not connect / bad host / bad port

# Handle host and port if non-default
if [ ! -z "$ZBX_REQ_DATA_URL" ]; then
URL="$ZBX_REQ_DATA_URL"
else
URL="$NGINX_STATUS_DEFAULT_URL"
fi

# save the nginx stats in a variable for future parsing
NGINX_STATS=$($WGET_BIN -q $URL -O - 2> /dev/null)

# error during retrieve
if [ $? -ne 0 -o -z "$NGINX_STATS" ]; then
echo $ERROR_DATA
  exit 1
fi

#
# Extract data from nginx stats
#
case $ZBX_REQ_DATA in
  active_connections) echo "$NGINX_STATS" | head -1 | cut -f3 -d' ';;
  accepted_connections) echo "$NGINX_STATS" | grep -Ev '[a-zA-Z]' | cut -f2 -d' ';;
  handled_connections) echo "$NGINX_STATS" | grep -Ev '[a-zA-Z]' | cut -f3 -d' ';;
  handled_requests) echo "$NGINX_STATS" | grep -Ev '[a-zA-Z]' | cut -f4 -d' ';;
  reading) echo "$NGINX_STATS" | tail -1 | cut -f2 -d' ';;
  writing) echo "$NGINX_STATS" | tail -1 | cut -f4 -d' ';;
  waiting) echo "$NGINX_STATS" | tail -1 | cut -f6 -d' ';;
  *) echo $ERROR_WRONG_PARAM; exit 1;;
esac

exit 0

root@local:~# cat /etc/zabbix/scripts/nginx/rc.sh
#!/bin/bash

# UserParameter=rc[*],/etc/zabbix/scripts/nginx/rc.sh "$1" "$2" "$3"

# Zabbix requested parameter
# item
ZBX_REQ_DATA="$1"
# domain
ZBX_REQ_DOMAIN=$(echo $2 | tr "/" ",")

ZBX_REQ_DATA_URL="$3"

# Nginx defaults
NGINX_STATUS_DEFAULT_URL="http://127.0.0.1:60913/rc"
WGET_BIN="/usr/bin/wget"

#
# Error handling:
# - need to be displayable in Zabbix (avoid NOT_SUPPORTED)
# - items need to be of type "float" (allow negative + float)
#
ERROR_NO_ACCESS_FILE="-0.9900"
ERROR_NO_ACCESS="-0.9901"
ERROR_WRONG_PARAM="-0.9902"
ERROR_DATA="-0.9903" # either can not connect / bad host / bad port

# Handle host and port if non-default
if [ ! -z "$ZBX_REQ_DATA_URL" ]; then
URL="$ZBX_REQ_DATA_URL"
else
URL="$NGINX_STATUS_DEFAULT_URL"
fi

# save the nginx stats in a variable for future parsing
NGINX_STATS=$($WGET_BIN -q $URL -O - 2> /dev/null)

# error during retrieve
if [ $? -ne 0 -o -z "$NGINX_STATS" ]; then
echo $ERROR_DATA
  exit 1
fi

case $ZBX_REQ_DATA in
    bytes_in) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN"  | cut -f3 -d',' ;;
    bytes_out) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f4 -d',' ;;
    conn_total) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN"| cut -f5 -d',' ;;
    req_total) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f6 -d',' ;;
    http_2xx) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f7 -d',' ;;
    http_3xx) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f8 -d',' ;;
    http_4xx) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f9 -d',' ;;
    http_5xx) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f10 -d',' ;;
    http_200) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f16 -d',' ;;
    http_206) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f17 -d',' ;;
    http_302) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f18 -d',' ;;
    http_304) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f19 -d',' ;;
    http_403) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f20 -d',' ;;
    http_404) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f21 -d',' ;;
    http_416) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f22 -d',' ;;
    http_499) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f23 -d',' ;;
    http_500) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f24 -d',' ;;
    http_502) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f25 -d',' ;;
    http_503) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f26 -d',' ;;
    http_504) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" | cut -f27 -d',' ;;
    http_other) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" |  cut -f28 -d',' ;;
    http_ups_4xx) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" |  cut -f30 -d',' ;;
    http_ups_5xx) echo "$NGINX_STATS" | grep "$ZBX_REQ_DOMAIN" |  cut -f31 -d',' ;;
esac
exit 0
root@local:~#

---------------------------------------------------
root@local:~#cat /usr/local/tools/http5xx_ststus.sh
#!/bin/bash
ts=`date +%s`;
#获取当前所监听的所有域名（域名/IP）
#wget  -q 127.0.0.1:60913/rc -O - 2> /dev/null | awk -F',' '{print $1"/"$2}'
domains=$(wget  -q 127.0.0.1:60913/rc -O - 2> /dev/null | awk -F',' '{print $1"/"$2}')

#根据需要grep出需要上报的域名：
#domains=$(wget  -q 127.0.0.1:60913/rc -O - 2> /dev/null | awk -F',' '{print $1"/"$2}' | grep "xxxxx.com/" )

domain=$(wget  -q 127.0.0.1:60913/rc -O - 2> /dev/null | awk -F',' '{print $1"/"$2}' | tr "/" "," )


#获取http_5xx
#wget -q http://127.0.0.1:60913/rc -O - 2> /dev/null | grep "mts.benmu-health.com,10.32.132.115:80" | cut -f10 -d','

# save the nginx stats in a variable for future parsing
NGINX_STATS=$(wget  -q 127.0.0.1:60913/rc -O - 2> /dev/null)
# error during retrieve
if [ $? -ne 0 ]; then
curl --connect-timeout 20 -X POST -d "[{\"metric\": \"nginx_status\", \"endpoint\": \"$HOSTNAME\", \"timestamp\": $ts,\"step\": 60,\"value\": 0,\"counterType\": \"GAUGE\",\"tags\": \"\"}]" http://127.0.0.1:1988/v1/push
else
curl --connect-timeout 20 -X POST -d "[{\"metric\": \"nginx_status\", \"endpoint\": \"$HOSTNAME\", \"timestamp\": $ts,\"step\": 60,\"value\": 1,\"counterType\": \"GAUGE\",\"tags\": \"\"}]" http://127.0.0.1:1988/v1/push

if [ ! -z  "$NGINX_STATS" ]; then
for i in $domains
do
REQ_DOMAIN=$(echo $i | tr "/" ",")
http_5xx=$(wget -q http://127.0.0.1:60913/rc -O - 2> /dev/null | grep $REQ_DOMAIN | cut -f10 -d',')
#Metric=$i
EndPoint=$i

curl --connect-timeout 20 -X POST -d "[{\"metric\": \"http5xx_ststus\", \"endpoint\": \"$EndPoint\", \"timestamp\": $ts,\"step\": 60,\"value\": $http_5xx,\"counterType\": \"COUNTER\",\"tags\": \"project=http-monitor\"}]" http://127.0.0.1:1988/v1/push
done
fi

fi
root@local:~#
--------------------------------------------------------------------
设置crontab push到falcon-server
echo "*/1 * * * * root /bin/sh /usr/local/tools/http5xx_ststus.sh" > /etc/cron.d/http5xx_ststus
-----------------------------------------------------------------------
open-falcon设置expressions策略告警

each(metric=http5xx_ststus project=http-monitor)
----------------------------------------------------
设置nginx_status告警
nginx_status all(#3) == 0 then alarm()
