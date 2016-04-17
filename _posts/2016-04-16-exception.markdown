---
layout: post
title:  "Java 异常处理实践"
date:   2016-04-16 16:58:10 +0800
categories: 工程实践
---
## 使用异常的目的与使用场景
异常情形是指阻止当前方法或者作用域继续执行的问题。把异常情形与普通问题相区分很重要，所谓普通的问题是指，在当前环境下能够得到足够的信息，总能处理这个错误。而对于异常情形，就不能继续下去了，因为在当前环境下无法获得必要的信息来解决问题。你所能做的就是从当前环境跳出，并且把问题提交给上一级环境。异常最重要的一个方面就是如果发生问题，它们将不允许程序沿着其正常的路径继续下去。

实际上，异常处理的一个重要目标就是把错误处理的代码同错误发生的地点相分离，这使你能在一段代码中专注于要完成的事情，至于如何处理错误，则放在另外一段代码中完成。这样一来，主干代码就不会与错误处理逻辑混在一起，也更容易理解与维护。

在我们实践中，我们把正常业务逻辑之外的业务异常也通过Java 异常机制来处理传递。我们以用户取钱这个动作为例，正常的业务逻辑代码只处理正常的情形，即用户顺利的取到了钱，至于用户余额不足，或者超出每天限额等业务异常情况都是通过抛出异常进行处理的，这些异常处理的代码都不在业务逻辑代码中，从而确保业务逻辑代码清晰，容易维护。

如果在业务逻辑中处理这些异常情况，我们就需要考虑如何在返回值中把异常情况返回给调用者，比如正常的取钱的方法的返回值时布尔类型，它是无法区分各种异常情况的，代码如下：

不包含异常处理的业务代码，代码结构非常清晰：
{% highlight java %}
public boolean withdraw(int userId, int amount) {
    //从用户账户取钱

    return true;
}
{% endhighlight %}

## 异常分类
Java中有2中方式定义异常，一种为每个异常情形定义一个异常，这种方案是会定义大量的异常，实际操作比较繁琐。另外一种方案是定义异常码，即为每一种异常情形定义一个异常码，这样做自定义异常类非常少，使用比较简单，缺点是不能根据异常类的名字确定异常情形。

我们在实践中摸索出使用异常码结合自定义异常的方式使用异常，具体来说我们自定义2个异常类，BusinessException代表业务异常，SystemException代码系统异常。业务异常主要是指业务逻辑上的异常，例如取钱时余额不足就是一个业务异常。系统异常主要包含IOException等系统级别的异常情形。

代码如下：

{% highlight java %}
public class BusinessException extends Exception {
    private int code;

    public BusinessException(String message, int code) {
        super(message);
        this.code = code;
    }
}
{% endhighlight %}

{% highlight java %}
public class SystemException extends Exception {
    private int code;

    public SystemException(String message, Throwable cause, int code) {
        super(message, cause);
        this.code = code;
    }
}
{% endhighlight %}

这样我们在业务逻辑代码中将所有的业务异常情形以BusinessException抛出来，将系统异常捕获住然后转换成SystemException异常抛出来，最后我们在最外层代码中捕获处理这两种异常即可。在包装SystemException时需要将异常链的信息带入进去。

## 异常码异常消息的处理
TODO

## 处理异常时打印异常日志
打印异常日志的原则是在最终捕获处理异常的地方使用Log4j进行记录异常日志，方便排障，而不会在抛出异常的地方打印异常日志，防止重复记录。也就是说打印日志通常是在最外层代码处理，而不会在业务逻辑代码中记录。在业务逻辑代码中捕获其它异常时(IOException等异常)时，我们只是把该异常包装到SystemExption中重新抛出，此时也不会记录异常日志。

## 异常处理原则
### 不要吃掉异常
对于自己不能处理的异常，要么不要捕获，在异常声明中声明，通知上级环境处理。要么自己捕获后进行包装，然后重新抛出，例如我们将IOException异常捕获后包装成SystemException重新抛出。千万不要把异常捕获住然后什么都不做，这样异常就丢失了。

### 不要在try块中放入大量的代码
一些新手常常把大量的代码放入单个try块，然后再在catch语句中声明Exception，而不是分离各个可能出现异常的段落并分别捕获其异常。这种做法为分析程序抛出异常的原因带来了困难，因为一大段代码中有太多的地方可能抛出Exception。

### 打印异常日志时尽量提供详细的上下文
在出现异常时，最好能够提供一些文字信息，例如当前正在执行的类、方法和其他状态信息，包括以一种更适合阅读的方式整理和组织printStackTrace提供的信息。尽量将详细的上下文写入异常日志中，这样当程序出问题时，查看异常日志进行定位错误时就会非常简单，例如可以将当前用户的userId等信息或者当前方法的主要输入参数记录下来，这样就可以方便定位错误。常见的错误做法是异常发生时打印“系统异常”等语句，排障无法获得任何有效信息。

### 不要使用异常来作代码的流程控制
异常处理机制性能开销时非常大的，因此尽量避免使用不必要的异常处理机制。在做很多操作时，我们其实是可以通过简单的判断来避免异常的抛出，例如除数为0时，数组或者ArrayList取值判断是否溢出，对象是否为null等情况，这样从而避免异常机制的介入，提升程序性能，同时代码的逻辑结构也更加清晰。下面的时一些例子：

错误用法：
{% highlight java %}
try{
  _map.put(myKey, myValue);
} catch(NullPointerException e){
  _map = new HashMap<String, String>();
}
{% endhighlight %}

正确用法：
{% highlight java %}
if(_map == null){
  _map = new HashMap<String, String>();
}
_map.put(myKey, myValue);
{% endhighlight %}

可以参考：[When is it OK to use exception handling for business logic?](http://stackoverflow.com/questions/5378005/when-is-it-ok-to-use-exception-handling-for-business-logic)

### 不指定具体的异常
尽量不要为了省事，就只捕获Exception类型的异常。

### 保证所有资源都被正确释放，充分运用finally关键词
一定牢记要finally中将异常发生后不再需要的资源释放掉，例如数据库链接、文件打开句柄等资源，否则将没有机会释放它们。
