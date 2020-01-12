# 深分页SQL如何优化？

### 问题现象

一个业务系统带有分页查询功能，但是随着查询页数的增加，越往后的查询性能越低，有时候一个查询需要一分钟左右的时间。分页查询的写法类似于：

``````sql
select * from employees limit 250000,30
``````

### 优化方案（2种）

###### SQL写法优化

```sql
select * from (select emp_no from employees limit 250000,30) a,employees b where a.emp_no = b.emp_no
```

###### 业务层优化

```sql
select * from employees where emp_no > #last_emp_no# order by emp_no limit 30
```

这种需要业务层配合，每次查询需要传递上次获取的最后一个id的值。

```2020-01-12```