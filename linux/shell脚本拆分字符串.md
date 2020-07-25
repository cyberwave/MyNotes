# 使用 awk 命令行分隔符截取字符串后赋值
* 场景
  在整 `fabric-ca` 时，需要将 `url` 拆分，并封装些内容。比如 `http://127.0.0.1:80` 添加用户名和密码。需要得到的效果如下：
  ```shell
  url=http://username:password@127.0.0.1:80
  ```
  `awk` 的默认分隔符是空格，可以使用 `-F` 添加想要根据某分隔符分隔的选项
  ```shell
  sampleUrl="http://127.0.0.1:80"
  protocol=`echo $sampleurl | awk -F "\/\/" {'print $1'}`
  hostPort=`echo $sampleUrl | awk -F "\/\/" {'print $2'}`
  resultUrl="$protocol//username:password@hostPort"
  ```
  上述的字符串可以从环境变量中获取值，例如 `sampleUrl=$HTTP_URL`
  这里使用 `-F` 将默认的空格分隔符替换为 `//`，文中对其进行了转义

# 将字符串分隔赋值给多个 shell 变量
例如字符串 `aaa|bbb|ccc` ，将其根据 `|` 进行分隔，并赋值给三个变量
```shell
#!/bin/bash
str="aaa|bbb|ccc"
OIFS=$IFS
IFS="|";
set -- $str
str1=$1;
str2=$2
str3=$3
IFS=$OIFS
echo $str1 $str2 $str3
```
or
```shell
#!/bin/bash
str="aaa|bbb|ccc"
OIFS=$IFS; IFS="|"; set -- $str; str1=$1; str2=$2; str3=$3; IFS=$OIFS
echo $str1 $str2 $str3
```
