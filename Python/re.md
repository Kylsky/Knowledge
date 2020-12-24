## 1.match(pattern, string, flags=0)

从字符串开头进行匹配，返回满足匹配条件的偏移量



## 2.fullmatch(pattern, string, flags=0)

将整个字符串进行匹配，字符串需要全匹配



## 3.search(pattern, string, flags=0)

和match差不多，但是不是从字符串开头进行匹配



## 4.sub(pattern, repl, string, count=0, flags=0)

对字符串进行截取，repl可以是字符串，也可以是函数

```python
#字符串
s = "123"
s = re.sub("\d","a",s)
print(s)

#函数
s = "A123B456"
def myMethod(matched):
	number1 = int(matched.group('value'))
	return str(number1 * 2)
s = re.sub('(?P<value>\d+)',myMethod,s)
print(s)
```



## 5.subn(pattern, repl, string, count=0, flags=0)

与sub差不多，不过返回的是字符串+替换次数的元组



## 6.split(pattern, string, maxsplit=0, flags=0)

```python
s = "A123B456C"
print(re.split('\d{3}', s))
```



## 7.findall(pattern, string, flags=0)

匹配所有满足条件的字符



## 8.finditer(pattern, string, flags=0)

查询出所有满足条件的字符串的迭代器

```python
s = '123A456B789C'
iterator = re.finditer('\d{3}', s)
for match in iterator:
    print(match.span())
```



## 9.compile(pattern, flags=0)

返回一个pattern对象，之后可以通过match匹配

```python
pattern = re.compile("\d+")
print(pattern.match('1234'))
```



## 10.purge()

清空缓存中的正则表达式

