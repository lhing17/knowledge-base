## jar命令
- 使用jar命令打包 ```jar -cvf test.jar . ``` 注意此命令打包spring boot的jar包会提示没有主清单属性
- 使用jar命令解压 ```jar -xvf test.jar ```
- 使用jar命令查看文件列表 ```jar -tvf test.jar ```
- 使用jar命令替换 ```jar -uvf test.jar XXX ```

## vim安装zip插件
可以使用vim的zip插件来替换jar包中的文本文件，但是有时会报错