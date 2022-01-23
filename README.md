# Github pages documentation powered by jekyll

## Test with 

Compile:

```
bundle exec jekyll build
bundle exec jekyll build --incremental --watch
```

Visit:

http://127.0.0.1:5500

## Assume docs folder unchanged during developement

cd docs/
git ls-files -z | xargs -0 git update-index --assume-unchanged

cd docs/
git ls-files -z | xargs -0 git update-index --no-assume-unchanged

## Layout documentation

https://mmistakes.github.io/minimal-mistakes/docs/layouts/

## Info about the images

identify *
convert -resize 50% img_unsplash_source_.jpg img_unsplash_source_res.jpg

## Git stage all but one folder

git add . && git reset -- docs

## Old docs

```
bundle exec jekyll serve --host=0.0.0.0
```
http://exp.ubuntu.local:4000/btcpayserver-docker/

