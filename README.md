# boucek.me
## Build & Deploy
- build: `hugo --gc --minify`
- gzip: `gzip -k -r **/*.html public/**/*.css public/**/*.xml public/**/*.txt public/**/*.jpg public/**/*.ttf public/**/*.woff2 public/**/*.eot public/**/*.svg public/**/*.woff`

## Deploy
- deploy to ~/www: `rclone sync public/ fishie:www/`
- then copy into `doas cp -R ~/www/* /var/www/htdocs/boucek.me/`

## Preview
- production-like: `hugo server`
- with drafts: `hugo -D server`
