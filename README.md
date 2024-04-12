# boucek.me
## Build & Deploy
- build: `hugo --gc --minify`
- deploy: `rclone sync public/ fishie:www/`
- deploy will copy stuff into /home/marian/www
- then copy into `doas cp -R ~/www/* /var/www/htdocs/boucek.me/`
## Preview
- production-like: `hugo server`
- with drafts: `hugo -D server`
