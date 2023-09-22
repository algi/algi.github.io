# boucek.me
## Build & Deploy
- build: `hugo --gc --minify`
- deploy: `rclone sync --interactive public/ boucek.me:www/`
- deploy will copy stuff into /home/marian/www
- then copy into `doas cp -R ~/www/* /jail/apache/usr/local/www/apache24/data/boucek.me/`
## Preview
- production-like: `hugo server`
- with drafts: `hugo -D server`
