# Manual for basic Magento2 speedup tweaks.
These are instructions for Hypernode. Instructions for Hipex would be a bit different. 
Jos


### varnish vcl:

1. upload tweaked vcl file to `/data/web/magento2`
2. `varnishadm vcl.load mag2 /data/web/magento2/vcl-jhp-optimized-jos.vcl`
3. `varnishadm vcl.use mag2`
4. `varnishadm vcl.list`  (see if new varnish vcl is used)
5. wait 5-6 minutes (see timestamp on `/data/var/varnish/default.vcl` <-- this needs to be updated)
6. `hypernode-servicectl restart varnish`
7. `hypernode-servicectl reload nginx`
8. check `varnishstat` for hit rate and cache usage.

### nginx buffers:

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

### Block bots (especially bingbot; GTFO):

1. Create extra file in: `/data/web/nginx` --> filename: `server.bots_goaway_jos`
2. add content:
```
if ($http_user_agent ~* (360Spider|bingbot|Adsbot|BLEXbot|SEOKicks|Mauibot|Riddler|ltx71|ZoominfoBot|seznam|velen|GrapeshotCrawler|Baidu|Censys|Pinterest) ) {
    return 410;
}
```
3. save
4. `hypernode-servicectl reload nginx`


### Optimize jpg images losslessly on server; in `/data/web/magento2/pub/media/` directory (except `/catalog` dir because of SRS import):
This also makes jpgs load progressive in browsers.

1. cd `/data/web/magento2/pub/media/`
2. `find . -type f -not -path "./catalog/*" -not -path "./tmp/*" -not -path "./import/*" -name "*.jpg" -exec jpegoptim --all-progressive -p -t -v -P {} \;`
3. let this run in the background. Can take a long time.

### magento backend:
1. stores > configuration > mirasvit > page cache warmer:
--> "Forcibly make pages cacheable"; set: configured + set 3 checkboxes; save config.

2. stores > configuration > advanced > system > full page cache: 
--> set TTL for public content to: 2629743 (1 month); save config.



====

### Really improve things:
- Upgrade Magento from 2.x naar 2.4.2, including ElasticSearch. ("Search_tmp table" bug); lower mysql load.
- Upgrade Template (Theme) to newest version.
- Improve SRS import script: 1. faster, import only changed items; 2. flush *only* cache for changed items, not complete FPC
- Remove unneeded JS and external pixels/scripts from template
- Confiure Varnish cache to use disk for FPC storage; 25GB+ (Not possible on Hypernode).
- Configure and use Cloudflare.

