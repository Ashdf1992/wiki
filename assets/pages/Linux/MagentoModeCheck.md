# Magento Cache & Mode Check
To check the Magento Cache and the mode that Magento is operating in, simply run the following:

```Bash
for i in cache:status deploy:mode:show; do php /var/www/vhosts/site.com/htdocs/bin/magento $i; done
```
