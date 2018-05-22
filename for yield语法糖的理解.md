# for yield语法糖的理解

## Example 1

```
for(x <- c1; y <- c2 ; z <- c3) {...}
```

被自动转换成

```
c1.foreach( x => c1.foreach( y => c2.foreach(z => {...})))
```

## Example 2

```
for(x <- c1;y <- c2 ; z <- c3) yield {...}
```

被自动转换成

```
c1.flatMap( x => c2.flatMap( y => c3.map( z => {...})))
```

## Example 3

```
for (x <- c;if cond) yield {...}
```

被自动转换为

```
c.withFilter( x => cond).map(x => {...})
```

## Example 4

```
for(x <- c;y = ...) yield {...}
```

被自动转化为

```
c.map(x => (x,...)).map((x,y) => {...})
```

为了加深理解有一个普通语法转成for yield语法

```
l.flatMap(sl=> slfilter(el=> el>0).map(el> el.toString.length))
```

转成for yield语法可以这样写

```
for{
    s1 <- l
    el <- sl
    if el > 0
}yield {
    el.toString.length
}
```



