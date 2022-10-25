# Flowable6 - 获得待办 | 字痕随行
原创 字痕随行 字痕随行

收录于话题

#流程引擎

55个

十年前，同时带三个项目，还能记清楚各种细节。十年后，记事本翻页，就已经记不清楚上一页内容了。

```Plain Text
花有重开日，人无再少年。
相逢拌酩酊，何必备芳鲜。
```
距离上一篇好长时间了，今天有点力气，简单说说Flowable的待办吧。

其实所谓的待办，归根到底就是通过查询act\_ru\_identitylink这张表得到的。

如果使用它的API接口来实现，简单的代码如下：

```Java
taskService.createTaskQuery().taskCandidateUser().list();
taskService.createTaskQuery().taskCandidateGroup().list();
taskService.createTaskQuery().taskAssignee().list();
```
但是使用这些接口来查询，太过理想化，很难适应复杂的业务场景。

如果使用自定义组（角色、部门、岗位等）分配办理人，然后查出当前登录人员所有的待办信息，并且还要分页展示的时候，用以上的接口就显得过于复杂。

所以，在单体项目中，将查询归根到数据库层面，通过自定义SQL查询是再好不过的。

比如角色，就可以使用下面的SQL进行查询：

```SQL
SELECT task_.*, execution_.BUSINESS_KEY_ FROM act_ru_task task_
INNER JOIN act_ru_execution execution_ ON task_.PROC_INST_ID_ = execution_.ID_
INNER JOIN act_ru_identitylink identitylink_ ON task_.ID_ = identitylink_.TASK_ID_
WHERE identitylink_.TYPE_ = 'ROLE' AND identitylink_.GROUP_ID_ IN (@roleIds)

```
如果想查询这个人属于各种自定义组的待办，就可以：

```SQL
SELECT task_.*, execution_.BUSINESS_KEY_ FROM act_ru_task task_
INNER JOIN act_ru_execution execution_ ON task_.PROC_INST_ID_ = execution_.ID_
INNER JOIN act_ru_identitylink identitylink_ ON task_.ID_ = identitylink_.TASK_ID_
WHERE identitylink_.TYPE_ = 'ROLE' AND identitylink_.GROUP_ID_ IN (@roleIds)
UNION
SELECT task_.*, execution_.BUSINESS_KEY_ FROM act_ru_task task_
INNER JOIN act_ru_execution execution_ ON task_.PROC_INST_ID_ = execution_.ID_
INNER JOIN act_ru_identitylink identitylink_ ON task_.ID_ = identitylink_.TASK_ID_
WHERE identitylink_.TYPE_ = 'DEPT' AND identitylink_.GROUP_ID_ IN (@roleIds)
```
通过自定义SQL进行查询，可以适应绝大多数业务场景，缺点就是需要熟悉Flowable的数据表结构。

上面的查询语句，只是解决了流程任务的查询问题。但是真正的待办，显示的是业务数据，我个人常用的有两种实现方法：

1.  每种业务数据通过不同的页面独立展示。这种实现方法，只需要在查询的时候关联指定的业务表即可。实现起来比较简单，但是待办无法集中展示。
2. 将每种业务数据的待办信息都抽象为通用的组成方式。比如待办都有：标题、创建时间、办理人、工单号等等。通过流程事件触发或其它拦截方式，在流程启动、任务创建时生成这部分信息，然后结合上面的查询SQL使用。这种实现起来较为复杂，而且怎么抽象是个问题，但是待办可以集中展示。

以上是单体项目的实现方式，分布式系统更加麻烦一些，我个人主要尝试过两种办法来实现：

1.  回写。在每个调用方存放一份副表，使用事件触发的方式，将流程数据回写，从而实现待办的查询，可以轻松的和业务数据合并展示。缺点也是同上，集中展示就无法实现了，而且极端情况下会遇到数据一致性问题。
2. 在流程服务中存储抽象后的通用信息。普通的待办可以集中读取，但是因为业务数据分布在自己的数据库中，所以根本无法进行联合查询。

到现在为止，关于流程待办的经验就这么多，欢迎分享和指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑