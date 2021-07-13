# Cloud-Native-DevSecOps
My personal blog

## Cloud Native

**For Cloud Native Blog Content**

## DevSecOps

**For DevSecOps Blog Content**

## Local debug

clone even theme code and copy them to `theme/even` dir
```
$ cd /tmp
$ git clone https://github.com/olOwOlo/hugo-theme-even themes/even
$ cp -rf themes/even Cloud-Native-DevSecOps/themes/even/
```

then run the below command to let hugo up
```
$ hugo server --bind 0.0.0.0 -p 9000
Start building sites …
hugo v0.85.0+extended darwin/amd64 BuildDate=unknown

                   | ZH-CN
-------------------+--------
  Pages            |    34
  Paginator pages  |     0
  Non-page files   |    10
  Static files     |    42
  Processed images |     0
  Aliases          |    12
  Sitemaps         |     1
  Cleaned          |     0

Built in 444 ms
Watching for changes in /Users/xiaomage/Downloads/Cloud-Native-DevSecOps/{archetypes,content,static,themes}
Watching for config changes in /Users/xiaomage/Downloads/Cloud-Native-DevSecOps/config.toml
Environment: "development"
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:9000/Cloud-Native-DevSecOps/ (bind address 0.0.0.0)
Press Ctrl+C to stop
```

Finally, service will be available `http://localhost:9000/Cloud-Native-DevSecOps/`.
