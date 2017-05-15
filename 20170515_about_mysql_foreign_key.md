mysql的初学者对于外键约束总会或多或少的产生一些疑惑,我把我测试的一些结果记录下来分享,如果有哪里说错了,希望得到大家的纠正



外键约束添加的测试通过以下两张表来完成:

```mysql
CREATE TABLE `category` (
  `c_cno` INT(11) NOT NULL AUTO_INCREMENT,
  `c_cname` VARCHAR(20) DEFAULT NULL,
  PRIMARY KEY (`c_cno`)
) ;

CREATE TABLE `product` (
  `p_pno` INT(11) NOT NULL AUTO_INCREMENT,
  `p_pname` VARCHAR(20) DEFAULT NULL,
  `p_ptype` INT(11) NOT NULL ,
  PRIMARY KEY (`p_pno`)
) ;
```

###添加没有给定名字的外键

先通过语句直接添加外键约束


```mysql
	ALTER TABLE product ADD FOREIGN KEY (p_ptype) REFERENCES category(c_cno);
```

现在外键添加成功了,我们怎么才能删除刚刚添加的外键呢?我在刚学习的时候,以为外键的名字就是p_ptype, 以为下面的语句可以删除添加的外键,结果会报错:
```mysql
	ALTER TABLE product DROP FOREIGN KEY p_ptype;
```
这是为什么呢?因为在添加外键的时候我们没有给这个外键设置名字,mysql会自己帮我们给他它一个默认的名字

通过下面操作可以查看到mysql给这个外键的默认名是什么:
```mysql
	SHOW CREATE TABLE product;

  product	CREATE TABLE `product` (
    `p_pno` INT(11) NOT NULL AUTO_INCREMENT,
    `p_pname` VARCHAR(20) DEFAULT NULL,
    `p_ptype` INT(11) NOT NULL,
    PRIMARY KEY (`p_pno`),
    KEY `p_ptype` (`p_ptype`),
    CONSTRAINT `product_ibfk_1` FOREIGN KEY (`p_ptype`) REFERENCES `category` (`c_cno`)
  ) ENGINE=INNODB DEFAULT CHARSET=utf8
```

我们可以看到被mysql设置的默认外键名是'product_ibfk_1';
```mysql
CONSTRAINT `product_ibfk_1` FOREIGN KEY (`p_ptype`) REFERENCES `category` (`c_cno`)
```
那既然知道了外键名,我们就能删除外键了,
```mysql
	ALTER TABLE product DROP FOREIGN KEY p_ptype;
```


### 添加有名字的外键约束

刚刚我们通过语句添加了没有给名字的外键约束,现在再来添加一个给了名字的外键约束:

```mysql
	ALTER TABLE product ADD CONSTRAINT fk_p2c FOREIGN KEY (p_ptype) REFERENCES category(c_cno);
```

上面的语句是说,我们给这个外键约束命名,命名为:'fk_p2c',想要删除外键就直接用模板语句套用外键名即可
```mysql
	ALTER TABLE product DROP FOREIGN KEY fk_p2c;
```



### 可以给同一个列多次添加外键约束
```mysql
ALTER TABLE product ADD CONSTRAINT fk_p2c1 FOREIGN KEY (p_ptype) REFERENCES category(c_cno);
ALTER TABLE product ADD CONSTRAINT fk_p2c2 FOREIGN KEY (p_ptype) REFERENCES category(c_cno);
ALTER TABLE product ADD CONSTRAINT fk_p2c3 FOREIGN KEY (p_ptype) REFERENCES category(c_cno);
```

这样做也是允许的,不过我也不知道为什么mysql要允许使用者这么用,我觉得这样没什么意义,如果有知道的请教我一下吧!

### 添加外键约束的一些注意点

1.一张表不能有重复的外键名

2.如果想让一个存在数据的外键关联一个已经存在的主键,那么外键列中的值不能超出主键列中的值

