# 数据库

1. ### 三大范式:

   1. 第一范式(确保每列的原子性)：每个列都不可以再拆分。
   2. 第二范式(确保每列都和主键相关)：在第一范式的基础上，非主键列全部要完全依赖于主键，而不能是依赖于主键的一部分。所有非主属性都完全依赖于R的每一个候选关键属性, **也就是说在一个数据库表中，一个表中只能保存一种数据，不可以把多种与主键无关的数据保存在同一张数据库表中。**
   3. 第三范式(**确保每列都和主键列直接相关,而不是间接相关, 去除冗余**)：在第二范式的基础上，非主键列只直接依赖于主键，不依赖于其他非主键(不存在传递依赖。)

   在设计数据库结构的时候，要尽量遵守三范式，如果不遵守，必须有足够的理由。比如性能。事实上我们经常会为了性能而妥协数据库的设计。

   ### 