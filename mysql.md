# mysql

## 行锁
  :record lock
  ```
  select * from a where id= 1 for update
  ```
  :next-key lock
```
update a set b=1 where id>10
```
  
  :gap lock
```
update a set b=1 where id>2 and id<50
```

## 表锁



