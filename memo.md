### Numerics

### Reflection

System.Type
抽象クラス

★Type は抽象であり、これから様々な方法で取得する Type は Type クラスを継承した子のものである。

typeof(int)で int の型情報(Type 型)が取得できる。
Type 型では、GetMethods,GetProperties,GetFields などのメソッドが使用できる。

```cs
Type t = typeof(int);
var methods = t.GetMethods();
var properties = t.GetProperties();
var fields = t.GetFields();
```

GetType()で type を取得することも可能。
この GetType は全てのオブジェクトで使用できるオブジェクト。

```cs
Type t2 = "hello".GetType();
```

Type 情報を取得できれば、アセンブリ情報も取得できる。
アセンブリは Type から構成され、アセンブリ内の Type 情報は GetTypes()で簡単に取得できる。

{System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e}

```cs
var a = typeof(string).Assembly
var types = a.GetTypes();

var fullName = types[10].FullName
"Interop+Kernel32+FINDEX_SEARCH_OPS"
```

GetTypes ではなく、GetType()ではタイプの完全修飾名(以下の例では System.Int64)を引数にタイプ情報を取得できる。

```cs
var a = typeof(string).Assembly
var types = a.GetTypes();

var fullName = types[10].FullName
"Interop+Kernel32+FINDEX_SEARCH_OPS"

var t = Type.GetType("System.Int64");
//当然t.FullNameはSystem.Int64となる。
```

#### Generics

List<T>の T の情報を type から取得する方法

```cs
var t = Type.GetType("System.Collections.Generic.List`1");
```

#### リフレクションを利用してインスタンスを作る

リフレクションからインスタンスを作る為に、アクティベーターというクラスが存在する。

```cs
var t = typeof(bool);
var b = Activator.CreateInstance(t);
var b2 = Activator.CreateInstance<bool>();
```

#### Delegate

var handlerMethod = demo.GetType().GetMethod("Handler");
Handler というメソッドが一つしかないので、引数の数を指定しなくても問題ない。

デリゲートを行っている箇所。
https://docs.microsoft.com/ja-jp/dotnet/api/system.delegate.createdelegate?view=net-5.0

```cs
var handler = Delegate.CreateDelegate(
  eventInfo.EventHandlerType,
  null, // object that is the first argument of the method the delegate represents
  handlerMethod
);
```

```cs
using System;

namespace Reflection
{
  public class Demo
  {
    public event EventHandler<int> MyEvent;

    public void Handler(object sender, int arg)
    {
      Console.WriteLine($"I just got {arg} from {sender?.GetType().Name}");
    }

    public static void Main_(string[] args)
    {
      var demo = new Demo();

      var eventInfo = typeof(Demo).GetEvent("MyEvent");
      var handlerMethod = demo.GetType().GetMethod("Handler");

      // we need a delegate of a particular type
      var handler = Delegate.CreateDelegate(
        eventInfo.EventHandlerType,
        null, // object that is the first argument of the method the delegate represents
        handlerMethod
      );
      eventInfo.AddEventHandler(demo, handler);

      demo.MyEvent?.Invoke(null, 312);
    }
  }
}
```

#### Attribute

public class RepeatAttribute : Attribute

〇〇 Attribute というクラスは、Attribute クラスを継承できる。
RepeatAttribute 属性を付与したいメソッドやプロパティに対して、[Repeat]or[RepeatAttribute]と書けば属性が付与される。
※コンストラクタで引数を取る場合には、[Repeat(1)]などと書く。

GetMethod()の戻り値の MethosInfo クラスが継承している MemberInfo クラスは GetCustomAttributes()を持っているので、
ここで Member の

```cs
using System;

namespace Reflection
{
  public class RepeatAttribute : Attribute
  {
    public int Times { get; }

    public RepeatAttribute(int times)
    {
      Times = times;
    }
  }

  public class AttributeDemo
  {
    [RepeatAttribute(3)]
    public void SomeMethod()
    {

    }

    public static void Main(string[] args)
    {
      var sm = typeof(AttributeDemo).GetMethod("SomeMethod");
      foreach (var att in sm.GetCustomAttributes(true))
      {
        Console.WriteLine("Found an attribute: " + att.GetType());
        if (att is RepeatAttribute ra)
        {
          Console.WriteLine($"Need to repeat {ra.Times} times");
        }
      }
    }
  }
}
```

### Dynamic Programming

### Extension Methods

### Memory Management
