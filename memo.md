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

動的に型を決められる事を late binding という。

使い方

```cs
using System;
using System.Dynamic;

namespace Dynamic
{
    public class Widget : DynamicObject
    {

    }

    class Demo
    {
        static void Main(string[] args)
        {
            dynamic w = new Widget();
            var w = new Widget() as dynamic;
        }
    }
}
```

#### Dynamic for XML Parsing

通常以下のような XML 解析を、

> node.Element("person").Attribute("name");

このようにシンプルに行いたい。

> xml.person.name;

```cs
using System;
using System.Dynamic;
using System.Text;
using System.Xml.Linq;

namespace Dynamic
{
    public class Demo
    {
        static void Main(string[] args)
        {
            var xml = @"
<people>
<person name='HANAKO'/>
</people>
";
            var node = XElement.Parse(xml);
            var name = node.Element("person").Attribute("name");
            Console.WriteLine(name?.Value);
        }
    }
}

```

#### Visitor Pattern with Dynamic Dispatch

以下の例では、addition.Left が Literal かどうか分からない為、
Print(addition.Left, sb)はエラーになる。

```cs
using System;
using System.Dynamic;
using System.Text;

namespace Dynamic
{
    public abstract class Expression
    {
        //Expressionクラスに何も手を加えずに、どのようにVisitorを迎え入れるか。
        //public abstract void Print(StringBuilder sb);
    }

    public class Literal : Expression
    {
        public double Value { get; }

        public Literal(double value)
        {
            Value = value;
        }
    }

    public class Addition : Expression
    {
        public Expression Left { get; }
        public Expression Right { get; }

        public Addition(Expression left, Expression right)
        {
            Left = left;
            Right = right;
        }
    }

    public class ExpressionPrinter
    {
        public static void Print(Literal literal, StringBuilder sb)
        {
            sb.Append(literal.Value);
        }

        public static void Print(Addition addition, StringBuilder sb)
        {
            sb.Append('(');
            /* addition.Leftはliteralではない為、変換できない。かといって、
            Print(Expression e, StringBuilder sb)のようなmethodを作り、内部で分岐するのもばかげている。*/
            Print(addition.Left, sb);
            sb.Append('+');
            Print(addition.Right, sb);
            sb.Append(')');
        }

        //ダメな例
        //public static void Print(Expression e, StringBuilder sb)
        //{
        //    if(e.GetType() == typeof(Literal))
        //    {
        //        Print((Literal)e, sb);
        //    }
        //}
    }

    public class DynamicVisitorDemo
    {
        static void Main(string[] args)
        {
            //1+2+3
            var e = new Addition(new Addition(new Literal(1), new Literal(2)), new Literal(3));
            var sb = new StringBuilder();
            ExpressionPrinter.Print(e, sb);
            Console.WriteLine(sb);
        }
    }
}

```

> Print((dynamic)addition.Left, sb);

(dynamic)を付け加えることで、実行時に型を判定し、自動で addition.Left が Literal 型なら Print(Literal literal, StringBuilder sb)が自動で呼び出される。
適切なオーバーロードが無ければ、ランタイムエラーが発生する。

コードの記述に関しては柔軟性がある一方で、同時にパフォーマンスが低下する。

Unhandled exception. Microsoft.CSharp.RuntimeBinder.RuntimeBinderException: The best overloaded method match for 'Dynamic.ExpressionPrinter.Print(Dynamic.Addition, System.Text.StringBuilder)' has some invalid arguments
at CallSite.Target(Closure , CallSite , Type , Object , StringBuilder )
at System.Dynamic.UpdateDelegates.UpdateAndExecuteVoid3[T0,T1,T2](CallSite site, T0 arg0, T1 arg1, T2 arg2)
at Dynamic.ExpressionPrinter.Print(Addition addition, StringBuilder sb) in C:\Users\KOHEI\source\repos\Dynamic\Program.cs:line 47
at System.Dynamic.UpdateDelegates.UpdateAndExecuteVoid3[T0,T1,T2](CallSite site, T0 arg0, T1 arg1, T2 arg2)
at Dynamic.ExpressionPrinter.Print(Addition addition, StringBuilder sb) in C:\Users\KOHEI\source\repos\Dynamic\Program.cs:line 47
at Dynamic.DynamicVisitorDemo.Main(String[] args) in C:\Users\KOHEI\source\repos\Dynamic\Program.cs:line 70

### Extension Methods

Reflection との組み合わせで効果を発揮する。

拡張メソッドを用いることで、既存のクラスに手を加えることなく処理を追加できる。
sealed などで継承できないクラスに手を加える場合に必要。

ExtensionMethods という静的なクラスの中に、静的なメソッドを追加していく。

```cs
using System;

namespace Extension
{
    public sealed class Foo
    {
        public string Name => "Foo";
    }

    public static class ExtensionMethods
    {
        public static int Measure(this Foo foo)
        {
            return foo.Name.Length;
        }
    }
    public class Demo
    {
        public static void Main()
        {
            new Foo().Measure();
        }
    }
}
```

作成したクラスだけでなく、ビルトインタイプにも手を加えられる。

```cs
using System;

namespace Extension
{
    public static class ExtensionMethods
    {
        public static string ToBinary(this int n)
        {
            return Convert.ToString(n, 2);
        }
    }
    public class Demo
    {
        public static void Main()
        {
            Console.Write(42.ToBinary());
        }
    }
}
```

ISerializable などのインターフェースでも拡張可能。
これでシリアル化可能なオブジェクトから利用可能なメソッドができる。

```cs
public static class ExtensionMethods
{
    public static void Save(this ISerializable serializable)
    {
        //...
    }
}
```

拡張 method は、ポリモーフィズムは考慮しない。
virtual クラスであっても、そのクラスの拡張 method が呼ばれる。

```cs
using System;
using System.Runtime.Serialization;

namespace Extension
{
    public class Foo
    {
        public virtual string Name => "Foo";
    }
    public class FooDerived : Foo
    {
        public override string Name => "FooDerived";
    }

    public static class ExtensionMethods
    {
        public static int Measure(this Foo foo)
        {
            return foo.Name.Length;
        }
        public static int Measure(this FooDerived fooDerived)
        {
            return 99;
        }
    }
    public class Demo
    {
        public static void Main()
        {
            var derived = new FooDerived();
            Foo parent = derived;
            Console.WriteLine("As Foo class:" + parent.Measure());
            Console.WriteLine("As FooDerived class:" + derived.Measure());
        }
    }
}
```

ToString メソッドのようにオブジェクトから使用可能なクラスをオーバーライドして、foo.ToString()のように呼び出すことはできない。

```cs
namespace Extension
{
    public class Foo
    {
        public virtual string Name => "Foo";
    }
    public static class ExtensionMethods
    {
        //object
        public static string ToString(this Foo foo)
        {
            return "test";
        }
    }
    public class Demo
    {
        public static void Main()
        {
            var foo = new Foo();
            Console.WriteLine(foo.ToString());
            Console.WriteLine(ExtensionMethods.ToString(foo));
        }
    }
```

(string, int)といったデータや tuple、delegate に対しても拡張 method は作成可能。

```cs
using System;
using System.Runtime.Serialization;

namespace Extension
{
    public class Person
    {
        public string Name;
        public int Age;

        public override string ToString()
        {
            return $"{nameof(Name)}: {Name}, {nameof(Age)}: {Age}";
        }
    }
    public static class ExtensionMethods
    {
        public static Person ToPerson(this (string name, int age) data)
        {
            return new Person { Name = data.name, Age = data.age };
        }

        public static int Measure<T, U>(this Tuple<T, U> t)
        {
            return t.Item2.ToString().Length;
        }
    }
    public class Demo
    {
        public static void Main()
        {
            var me = ("HANAKO", 20).ToPerson();
            Console.WriteLine(me);

            //tuple
            Tuple.Create(1, false).Measure();
        }
    }
}
```

delegate に拡張 method を使用することも可能。

```cs
using System;
using System.Diagnostics;
using System.Runtime.Serialization;
using System.Threading;

namespace Extension
{
    public static class ExtensionMethods
    {
        public static Stopwatch Measure(this Func<int> f)
        {
            var st = new Stopwatch();

            st.Start();
            f();
            st.Stop();
            return st;
        }
    }
    public class Demo
    {
        public static void Main()
        {
            Func<int> calculate = delegate
            {
                Thread.Sleep(1000);
                return 42;
            };
            var st = calculate.Measure();
            Console.WriteLine($"Took {st.ElapsedMilliseconds}msec");
        }
    }
}
```

上で見てきたようにあらゆるタイプの拡張は以下の 2 つの方法で実行できる。

```cs
//パターン1
public static void Foo(this object o)
{

}
//パターン2
public static void Foo<T>(this T o)
{

}
```

任意のオブジェクトのディープコピーを行う場合は、以下のように設定する。

```cs
public static void DeepCopy<T>(this T o)
where T : ISerializable ,new()
{

}
```

#### 既存のクラスにデータを追加する方法

残念ながら、拡張プロパティのようなものは存在しない。
既存のクラスにデータを追加するには、以下のような方法を取る必要がある。

特定のオブジェクトにデータやタグを持たせる方法

```cs
using System;
using System.Collections.Generic;
using System.Linq;

namespace Extension
{
    public static class ExtensionMethods
    {
        private static Dictionary<WeakReference, object> data
            = new Dictionary<WeakReference, object>();

        //TagとはDictionary<WeakReference, object>のobject部分の事。
        public static object GetTag(this object o)
        {
            var key = data.Keys.FirstOrDefault(k => k.IsAlive && k.Target == o);
            return key != null ? data[key] : null;
        }

        public static void SetTag(this object o, object tag)
        {
            var key = data.Keys.FirstOrDefault(k => k.IsAlive && k.Target == o);
            if (key != null)
            {
                data[key] = tag;
            }
            else
            {
                data.Add(new WeakReference(o), tag);
            }
        }


        public class Demo
        {
            public static void Main()
            {
                string s = "Meaning of life";
                s.SetTag(42);
                Console.WriteLine(s.GetTag()); //42
            }
        }
    }
}
```

https://ufcpp.net/study/csharp/RmWeakReference.html

WeakReference
Represents a weak reference, which references an object while still allowing that object to be reclaimed by garbage collection.

### Memory Management
