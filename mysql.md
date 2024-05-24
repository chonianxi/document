# mysql

~~删除线~~
>引用块

## 行锁
  : **record lock**
  >锁住id=1的那一行
  ```
  select * from a where id= 1 for update
  ```
***
  : next-key lock
  >如果存在id>10的则是next-key lock
>>否则就是无锁
```
update a set b=1 where id>10
```
  ***
  : gap lock
```
update a set b=1 where id>2 and id<50
```

## 表锁
1. asfasdf
2. safdasfd
3. asfasdfsdf  


