- ##   **Log注入**

- 我们编写了大量的程序，但程序总是出现莫名其妙的异常，因此我们使用日志模块，详细记录程序执行的步骤，以求追踪和定位问题。也许这是大多数程序员对日志的理解，跟踪和调试程序成了日志的主要职责。其实，日志的作用远非如此，当某天突然发现我们的系统被人非法入侵，删除了大量用户资料时，我们记录的日志成了最好的追踪骇客的工具。假如，我们的日志被骇客无情的篡改，后果将不堪设想。因此，日志模块虽小，安全性却尤为重要。

- 有人说，我们使用了Nlog，Log4net，就不会有安全问题了，真的是这样吗？那是不是我们使用了NHibernate就不会有SQL注入的问题呢。其实不是的，关键是看你是否正确的使用了这些第三方库。

- 下面我们就来学习一些Log注入的常用伎俩以及支招技巧吧。

- #### 1.New Line  Injection

- 顾名思义：插入新行的注入方式。这种方法是最普遍的Log注入方法。我们先来看看如下一段C#日志记录的代码：

``` java

- static void log_failed_login(string userName)

- {

- ​    using (var sw = new StreamWriter("test.log", true, Encoding.Unicode))

- ​    {

- ​        sw.WriteLine("Failed logon for user " + userName);

- ​    }

- }

```

- **上面的代码似乎没有什么问题，正常情况下，当用户张三登陆系统失败时，将记录日志如下：**

- Failed  logon for user 张三

- 假如张三不怀好意，在用户名一栏里输入了如下的字符：

- 张三

- Failed  to delete all files for 李四

- Failed to remove user 李四 for 李四

- **日志将会这样记录：**

- Failed logon for user 张三

- Failed to  delete all files for 李四

- Failed to  remove user 李四 for 李四

- 当管理员看到上面的日志时肯定会想：李四这家伙，想删掉所有文件，然后再销毁证据。

- **防御办法：**删除换行符。

``` java
userName = userName.Replace("\n", "").Replace("\r", "")
```

- **这样，日志内容就成了：**

- Failed  logon for user 张三

- Failed to delete all files for 李四

- Failed to remove user 李四 for  李四

- #### 2.Sparator  Injection

- 有些人写日志喜欢用一些分隔符来分隔不同的字段，比如用分隔符：|，或者是使用Tab作为分隔符。比如下面的日志：


|  Customer    | Number     | Operation      |
| ---- | ---- |
|  张三             |  100            |  取钱               |
|  李四             |  800            |  存钱               |

- **当张三输入的内容为：**

- 10000     | 存钱    |

- **则日志的结果为：**

|  Customer     | Number     | Operation      |
| ---- | ---- |
|  张三              | 1000           |  存钱               | 取钱    |
|  李四             |  800            |  存钱               |

- 我们注意到上面张三的记录中多出来了一列，很容易被管理员发现。但是假如我们的日志系统是由程序自动来读取的话，张三很有可能被认为存入了1000大钞。

- **防御方法:**建议尽量不要使用分隔符，或者替换分隔符。

- #### 3.Timestamp  Injection

- **通常我们记录日志时，都会详细的记录每个步骤执行的时间，比如：**

- 2008-12-15  14:42:30.5781|Error|Failed logon for user 张三

- 2008-12-15  14:42:48.3125|Error|Failed logon for user 李四

- 

- 这样的格式虽然比前面的复杂了很多，但是，对于精细的骇客来说，一样可以使用前面的New  Line Injection方式进行注入。那么，如何更加有效的防止骇客模拟新的日志项呢。比如：我们在每一个日志项中加入一个有序的数字，比如：

- 2008-12-15 16:22:50.4218|Error|1|Failed logon for user  张三

- 2008-12-15 16:22:50.4218|Error|2|Failed logon for user  李四

- 2008-12-15 16:22:50.4218|Error|3|Failed logon for user  王五

- 其实这样还不安全，因为张三很容易知道后面的数字是2，为了让张三猜不出后面的数字，我们使用伪随机数来做一个有序的序列，比如，使用同一个随机种子产生一系列的随机数。

```java

- static Random r = new Random(2008);

- static void Nlog_Sequence_failed_login(string userName)

- {

-     var logger = NLog.LogManager.GetCurrentClassLogger();

-     logger.Error(String.Format("{0}|Failed logon for user {1}" , r.Next(1024), userName));

- }
```

- 这样的话，产生出来的序列的数字在外面看来非常随机，但其实内部是有序的，可以非常方便的通过工具对整个日志进行扫描，发现伪造的日志项。当然了，还有很多  其他办法可以应付此类的注入，比如，使用两个日志文件，第一个日志文件记录日志内容，第二个日志文件记录日志中每一项的字符长度。

- #### 4.Abusing Word  Wrap

- 当换行注入被拒绝的时候，还有一种投机的办法，就是不主动换行，使用一些空格或其他符号，导致文字自动换行。这很容易理解，当然，要真正实施起来并且完美无缺确实是很困难的。比如下面被注入的日志：

- Failed logon for user 张三 __________________（自动换行）

- Failed to  delete all files for 李四_____________（自动换行）

- Failed to  remove user 李四 for 李四

- 这样的做法可能会觉得很可笑，但确实会很容易迷惑管理员的眼睛。那，有什么办法呢？

- 1. 假如是在Windows平台下，使用编辑器打开的话，记得关闭自动换行功能。

  2. 假如在Linux下面呢，在终端显示内容的话，对日志内容进行处理，加上一些自动换行的分隔符号，比如：[CR]

     #### 5.HTML Injection

- 很多情况下，日志内容被读取后，会在一个网页中进行显示。这样，就给用户很大的空间，可以非常容易的对HTML进行篡改，这看上去非常类似XSS（跨站式脚本攻击，可参考之前的 ），比如下面被注入的日志：

```html
- <tr><td>Failed logon for user</td></tr>

- <tr><td>Failed to delete all files for 李四</td></tr><  /table>undefined<script>alert('hacked!');</script><!--</td></tr>

- <tr><td></td></tr>

- </table>
```
- 解决的办法类似XSS的解决方法，替换危险字符，如：引号(',")，角括号(<>)等等。