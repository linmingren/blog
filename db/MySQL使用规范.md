   MySQL估计是大家日常最常用的数据库，我们开发的每个项目也都是用了MySQL， 下面是我们参考了一些资料还有自己项目的经验记录的一些使用规范。虽然这些规范都很简单，
 但是真正这么用的开发团队其实不多（尤其是没有积累的小十几，二十号人的小公司），按我们的经验来看，只要严格按这个规范作了，基本上支撑200W用户是没有问题的。当然如果
   
   谈到规范，先说一下数据量，其实笼统说什么千万数据量意义不大，因为表的数据不一样，像日志，流水这样的表， 其实就算上亿也无所谓的，因为这些数据和其他数据关联不大，
   在业务上用到它的机会也很少（可能就是查看流水时会查一下，或者出问题的时候会查一下），所以简单地把旧的日志和流水移动到历史表就行了。
   
### 使用规范
   * 必须使用InnoDB引擎。 这个看起来像白痴的一样的， 你可能会想这还用说吗，肯定是InnoDB啊。 不过我还是见过一些不用Inno，原因都是创建表的语句是从别的地方copy过来的，原来的可能不是InnoDB引擎的。
   * 必须使用utf8mb4编码， collation用utf8mb4-bin, 之所以用这个编码是为了保存表情类数据。尤其是collation一定不要选那些乱七八糟的比较方式。
   * 事务级别应该是读提交，因为我们的应用基本都是OLTP应用。
   * 如果表记录超过2000w， 就需要分表了。
   * 使用外键约束。 有很多指南建议你不要用外键，那是为了追求高性能作的妥协， 对我们常见的项目，数据量还远远达不到那个量级，所以用外键来约束关联是利大于弊。
   * 凡是浮点数都应该用decimal来保存。
   * 不要用uuid来做主键。 有些指南忽悠你用uuid来做主键， 就算你的数据量真的巨大，不想用数据库自增id，也应该用别的可以递增的id，绝对不要用uuid，我们曾经在一个项目里
   用过，简直是血泪史。
   * INT类型不需要写成INT(N)这种格式，我见过的10个开发人员， 有9个会认为里面的N是说这个字段只能保存这么大的整数。
   * 不要用union，而是用union all。
   * 非实时的统计，用定时任务定时更新就好了。 千万别上什么Flink之类的流计算引擎。
   * 流水表， 一定要记录清楚更改前后的状态和原因。 比如更改前的金额，更改后的金额， 因为哪个订单导致的金额修改。如果是一些分红类的流水表， 一定要记录当时的分红比例，
   分红级别这些信息，否则后续定位问题很难。
   * 涉及金额变动的操作，一定要设计一个“冲账”的类型， 当操作有问题（比如分红金额错误）时，可以把旧的分红记录給“冲”掉。这样统计起来就很清楚。
   * 对所有涉及到金额变动的表，都应该有一张对应的流水表。比如OTC订单， 一个订单会有很多笔成交单，每笔成交都会导致原始订单的金额出现变化， 需要在流水中表记录起来，否则
   出问题很难定位出来当前订单的余额是怎么出来的。
   * 不适用存储过程。 这里不适用存储过程的原因都不是它不能扩展， 而且它的调试和编写都很麻烦，一旦写这个存储过程的人离职了，后面接手的人可能根本看不懂存储过程是干嘛的。
   * 上线后，对数据库结构的改动，都需要记录在独立的文件，每个版本都保存起来。
   * 线上直接操作数据库前，切记先备份。 
   