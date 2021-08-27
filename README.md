# Manual for basic Magento2 speedup tweaks.
These are instructions for Hypernode. Instructions for Hipex would be a bit different. - Jos


### 1. enable large Mysql Thread stack on Hypernode
This might help with making sure that large / long-running scripts won't hang. 
One CLI command needed.

`hypernode-systemctl settings mysql_enable_large_thread_stack --value True`

After this, a MySQL restart is needed. Couple of minutes downtime.

For background info see here: https://support.hypernode.com/en/hypernode/mysql/how-to-configure-a-large-mysql-thread-stack





### 2. Check and increase RAM usage for MySQL (innodb_buffer_pool_size)

On Hypernode, mostly default settings are used for MySQL. This is sometimes OK for smaller sites. Most of the time: not optimal.
Hypernode can and will change these settings on request per mail.

#### A: check with tuning-primer
1. login via SSH
2. do `wget https://raw.githubusercontent.com/BMDan/tuning-primer.sh/master/tuning-primer.sh`
3. do `chmod +x tuning-primer.sh`
4. Run tuning-primer.sh by `./tuning-primer.sh` -- and look at the report and values.
5. Especially look for: `join_buffer_size`, `table_open_cache`, `table_definition_cache`, `table_cache`, and `innodb_buffer_pool_size`.

#### B: make note of settings and email Hypernode
On a small database, the `innodb_buffer_pool_size` is mostly OK. On a larger database, if not large enough, swaps will occur between memory and disk.
This value needs to be increased. A normal "rule of thumb" is that mysql memory should take up about 80% of total RAM.
I advise that we use a mamximum of 50% of RAM, just to be sure that there is more than enough memory left for nginx, elastic, varnish, php and redis.
1. Look at `htop` memory usage.
![image](https://user-images.githubusercontent.com/80516148/126620340-a490ad5e-0507-4d0d-ba85-78667b544774.png)
In this example you see that only ~10G of total 32G is used. We can safely increase RAM usage of MySQL to around 16G (if needed).
2. In this case we can ask Hypernode to increase the setting of `innodb_buffer_pool_size` to 16G.




### 3. varnish vcl:

1. upload [the tweaked vcl file](https://github.com/JosQlicks/magento-speed-tweaks/blob/main/vcl-jhp-optimized-jos.vcl) to `/data/web/magento2`
2. `varnishadm vcl.load mag2 /data/web/magento2/vcl-jhp-optimized-jos.vcl`
3. `varnishadm vcl.use mag2`
4. `varnishadm vcl.list`  (see if new varnish vcl is used)
5. wait 5-6 minutes (see timestamp on `/data/var/varnish/default.vcl` <-- this needs to be updated)
6. `hypernode-servicectl restart varnish`
7. `hypernode-servicectl reload nginx`
8. check `varnishstat` for hit rate and cache usage.




### 4. nginx buffers:

1. Create extra file in: `/data/web/nginx` --> filename: `server.header_buffer_jos`
2. add content:
```
fastcgi_buffers 16 16k;
fastcgi_buffer_size 32k;
proxy_buffer_size 128k;
proxy_buffers 4 256k;
proxy_busy_buffers_size 256k;
```
3. save
4. `hypernode-servicectl reload nginx`



### 5. Block bots (especially bingbot; GTFO):

1. Create extra file in: `/data/web/nginx` --> filename: `server.bots_goaway_jos`
2. add content:
```
if ($http_user_agent ~* (360Spider|bingbot|BLEXbot|SEOKicks|Mauibot|Riddler|ltx71|ZoominfoBot|seznam|velen|GrapeshotCrawler|Baidu|Censys|Pinterest) ) {
    return 410;
}
```
3. save
4. `hypernode-servicectl reload nginx`




### 6. Optimize jpg images losslessly on server with jpegoptim:
in the `/data/web/magento2/pub/media/` directory.
(except `/catalog` dir because of SRS import). This also makes jpgs load progressively in browsers.

1. `cd /data/web/magento2/pub/media/`
2. `find . -type f -not -path "./catalog/*" -not -path "./tmp/*" -not -path "./import/*" -name "*.jpg" -exec jpegoptim --all-progressive -p -t -v -P {} \;`
3. let this run in the background. Can take a long time.

### 7. Optimize PNG images losslessly on server with pngopt:
1. `cd /data/web/magento2/pub/media/`
2. `find -type f -iname "*.png" -exec optipng -o4 -v -preserve {} \; -exec touch -m -a  {} \; -exec chmod 755 {} \;`
3. let this run in the background. Can take a long time.


### 8. magento backend settings:
1. stores > configuration > mirasvit > page cache warmer:
--> "Forcibly make pages cacheable"; set: configured + set 3 checkboxes; save config.

2. stores > configuration > advanced > system > full page cache: 
--> set TTL for public content to: 2629743 (1 month); save config.





====


## Really improve things:
- Upgrade Magento from 2.x naar 2.4.2, including ElasticSearch. ("Search_tmp table" bug); lower mysql load.
- Upgrade Template (Theme) to newest version.
- Improve SRS import script: 1. faster, import only changed items; 2. flush *only* cache for changed items, not complete FPC
- Remove unneeded JS and external pixels/scripts from template
- Configure Varnish cache to use disk for FPC storage; 25GB+ (Not possible on Hypernode).
- Configure and use Cloudflare.

