**查看已安装字体:** fc-list

**没有mkfontscale命令:** yum install mkfontscale

```shell
cp -v TimesNewRomanPSMT24163565.ttf /usr/share/fonts/
 
cd /usr/share/fonts/
mkfontscale
mkfontdir
fc-cache -fv
 
fc-list
```

