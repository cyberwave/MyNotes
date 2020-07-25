# 数据库字段默认值
在数据库中，非 `createtime`、`updatetime`，的其他字段，定义为  `VARCHAR(20)`，默认值为 `DEFAULT ''`
在查询的时候，附加了这样一个条件 `date_format(temptime, '%Y-%m-%d %H:%i:%s')`, 由于该字段时间格式的内容
而导致查询出错。 将默认值设置为 `DEFAULT '0000-00-00 00:00:00'，在插入一条语句后，查询结果仍为空串（不是NULL）
原因可能是因为该字段被语言默认的零值给覆盖了。字符串的零值一般是一个空串。

# 处理方式一: 直接处理
由于该字段为 `VARCHAR`，可以不用处理，查询出的内容即为结果内容

# 处理方式二: 使用数据库函数处理
语法：
```sql
SELECT IF(IFNULL(fieldName,"默认字符串")="","默认字符串", fieldName) '重命令字段名' FROM tableName;
```
注意: 上述的 `默认字符串` 是自己设定的，`fieldName` 为数据库的字段名, `重命名字段名` 为将查询的字段名重命一个名字，否则
会是前面 `IF()` 一大串，一般重命名字段名为数据库当前字段的字段名，即上面的 `fieldName`。

`IF()` 函数是 `MySQL` 的内置函数，语法为：`IF(condition, valueOfTrue, valueOfFalse)`,类似一个三元的运算， `condition` 为必选，后面两值为可选。如果 `condition` 为真，则结果为第二个值，否则为第三个值。
`IFNULL()` 函数的语法为: `IFNULL(expression, alt_value)` , 判断第一个表达式是是否为 `NULL`，如果是，则返回第二个参数的值，否则返回第一个参数的值。
上面语法的意思是：将 `IFNULL(fieldName, "默认字符串")` 的结果为空串比较，如果为真（IFNULL()的结果是空串），则 `fieldName` 的值
为默认字符串，否则为 `fieldName` 的值。
