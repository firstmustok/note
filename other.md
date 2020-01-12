## go get 和git使用socks5代理

[https://blog.csdn.net/idwtwt/article/details/84842361]()
### go get：
```shell
http_proxy=socks5://127.0.0.1:1080 go get -u github.com/gocolly/colly/...
```

### git：
```shell
git config --global http.proxy socks5://127.0.0.1:1080
```