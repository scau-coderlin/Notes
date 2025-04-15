# 添加环境变量

## 临时

```shell
export UNIONREC=PATH # UNIONREC: 环境变量名，Path: 路径
```

## 永久

```sh
# 1. 创建 .bash_profile 文件
touch ~/.bash_profile

# 2. 添加以下内容并保存
export UNIONREC=Path # UNIONREC: 环境变量名，Path: 路径

# 3. 使文件生效
source ~/.bash_profile
```



# 查看文件md5值

```sh
md5sum filename # linux
csum filename # aix
certutil -hashfile filename MD5 # windows
```



# 压缩

## tar

```sh
tar -zcvf package_name.tar.gz dir/
```



# 解压

## unzip

```sh
# 解压到指定文件夹, destdir
unzip name.zip -d destdir
```



## tar

```sh
# 解压到指定文件夹, destdir
tar -zxvf name.tar.gz -C destdir
```



