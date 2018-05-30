# slick开发遇到的实际问题

*其他的比较常见的问题可以参考之前[Slick example](Slick example.md)的说明*

[参考来源](http://www.it610.com/article/3589853.htm)

## 1、分页查询

```scala
val userList = table.filter(_.userId === 0).sortBy(_.id).drop(page * size).take(size)
```

**page代表第几页，size则代表每页有几条数据**

## 2、模糊查询

```scala
val userList = table.filter(_.name like "%"+张+"%").sortBy(_.uid)
```

## 3、更新数据

```scala
table.filter(_.uid === uid).filter(_.tombstone === 0)
    .map(row => (row.password, row.updtime))
        .update(("要修改的密码", "要修改的时间"))
```

## 4、删除数据

```scala
 val q = for {
      f <- userDemo
      if f.projectId === project_id 
    } yield {
      f
    }
    db.run(q.delete)
```

