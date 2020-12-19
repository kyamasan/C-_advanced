### Task

#### Creating and Starting Tasks

write メソッドがやりたいタスクで、別々のスレッドで実行したい。

Task.Factory を使うことで、タスクを作成できる。

```cs
using System;

namespace TPL
{
    class Program
    {
        public static void Write(char c)
        {
            int i = 1000;
            while (i -- > 0)
            {
                Console.WriteLine(c);
            }
        }
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
        }
    }
}
```

タスク開始時には 2 つの方法がある。
1 つ目はアクションを Action 型の引数を渡す方法。
Action は通常関数以外にも、匿名関数やラムダ式にすることができる。

2 つ目は Task を new する方法。
https://ufcpp.net/study/csharp/functional/fun_localfunctions/#anonymous-function

このプログラムを実行すると、別々のスレッドで.と?がそれぞれ 1000 回印字される。

```cs
static void Main(string[] args)
{
    //スレッド1
    Task.Factory.StartNew(() => Write('.'));

    //スレッド2
    var t = new Task(() => Write('?'));
    t.Start();

    //メインスレッド
    Write('-');

    Console.WriteLine("Main program done.");
}
```

ラムダ式以外に以下のような方法で Action を記述できる。

```cs

public static void Write(object o)
{
    int i = 1000;
    while (i-- > 0)
    {
        Console.Write(o);
    }
}

static void Main(string[] args)
{
    Task t = new Task(Write, "hello");
    t.Start();
    Task.Factory.StartNew(Write, 123);

    Console.WriteLine("Main program done.");
    //ユーザの入力を待つ
    Console.ReadKey();
}
```

Task に値を渡し、受け取る方法(ジェネリック)

task1.Result のように書くと、タスクの結果を取得するまでその後の処理をブロックする。
このような処理をブロック処理と呼ぶ。

```cs
using System;
using System.Threading.Tasks;

namespace TPL
{
    class Program
    {
        public static int TextLength(object o)
        {
            Console.WriteLine($"Task with id {Task.CurrentId} processing object {o}...");
            return o.ToString().Length;
        }

        static void Main(string[] args)
        {
            string text1 = "testing";
            string text2 = "this";

            var task1 = new Task<int>(TextLength, text1);
            task1.Start();

            Task<int> task2 = Task.Factory.StartNew(TextLength, text2);

            //Resultを書くことで、処理完了まで待つことができる。
            Console.WriteLine($"Length of {text1} is {task1.Result}");
            Console.WriteLine($"Length of {text2} is {task2.Result}");

            Console.WriteLine("Main program done.");
            //ユーザの入力を待つ
            Console.ReadKey();
        }
    }
}

```

#### Task Cancelling

キャンセレーショントークンを使用することで、処理を途中でストップすることができる。
break などの処理よりも、例外を投げる方法が推奨されている。

OperationCanceledException()は他の例外と違い、処理が異常終了させて高レベルに例外を引き起こすことがない。

token.ThrowIfCancellationRequested();と書けば if 文が不要になるのでオススメ。

```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace TPL
{
    class Program
    {
        static void Main()
        {
            var cts = new CancellationTokenSource();
            var token = cts.Token;

            var t = new Task(() =>
            {
                int i = 0;
                while (true)
                {
                    if (cts.IsCancellationRequested)
                    　　//方法1.breakなど
                        break;
                        //方法2.例外
                        throw new OperationCanceledException();

                    else
                        Console.WriteLine($"{i++}");

                    //例外を投げるならif文を書かなくても良い。
                    token.ThrowIfCancellationRequested();
                    Console.WriteLine($"{i++}");
                }
            }, token);
            t.Start();

            Console.ReadKey();
            cts.Cancel();

            Console.WriteLine("Main program done.");
            Console.ReadKey();
        }
    }
}

```

token.Register メソッドは Cancel 実行後ただちに呼び出される関数。
token.WaitHandle.WaitOne()は token がキャンセルが実行されるまで処理を待機している。
どちらも、キャンセル実行をログとして残すことができる。

```cs
var cts = new CancellationTokenSource();
var token = cts.Token;

token.Register(() =>
{
    Console.WriteLine("Cancelation has been requested.");
});

Task.Factory.StartNew(() =>
{
    token.WaitHandle.WaitOne();
    Console.WriteLine("Wait handle released.");
});
```

タスクに大して CancellationToken を渡さないと、キャンセル後のタスクのステータスが変わってしまう。

```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace TPL
{
    class Program
    {
        static void Main()
        {
            //計画用キャンセレーショントークン
            var planned = new CancellationTokenSource();
            //予防用キャンセレーショントークン
            var prevenative = new CancellationTokenSource();
            //緊急用キャンセレーショントークン
            var emergency = new CancellationTokenSource();

            var paranoid = CancellationTokenSource.CreateLinkedTokenSource(
                planned.Token, prevenative.Token, emergency.Token);

            var t = new Task(() =>
            {
                int i = 0;
                while (true)
                {
                    paranoid.Token.ThrowIfCancellationRequested();
                    Console.WriteLine($"{i++}");
                    Thread.Sleep(1000);
                }
            }, paranoid.Token);
            t.Start();

            Console.ReadKey();
            emergency.Cancel();

            try
            {
                Task.WaitAll(t);
            }
            catch (AggregateException)
            {
                Console.WriteLine(t.Status);
            }
            Console.WriteLine("Main program done.");
            Console.ReadKey();
        }
    }
}

```

#### タスクの待機

```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace TPL
{
    class Program
    {
        static void Main()
        {
            var cts = new CancellationTokenSource();
            var token = cts.Token;
            var t = new Task(() =>
            {
                Console.WriteLine("I take 5 seconds");

                //1秒*5回待つ
                for (int i = 0; i < 5; i++)
                {
                    token.ThrowIfCancellationRequested();
                    Thread.Sleep(1000);
                }

                Console.WriteLine("I'm done");
            }, token);
            t.Start();

            Task t2 = Task.Factory.StartNew(() => Thread.Sleep(3000), token);

            //t(5秒)が完了するまで待つ⇒5秒後に"Main program done."と"I'm done"
            //t.Wait(token);
            //t(5秒)とt2(3秒)が両方完了するまで待つ⇒5秒後に"Main program done."と"I'm done"
            //Task.WaitAll(t, t2);
            //t(5秒)とt2(3秒)のどちらかが完了するまで待つ⇒3秒後に"Main program done."と5秒後に"I'm done"
            //Task.WaitAny(t, t2););
            //t(5秒)とt2(3秒)が両方完了するまで待つ(ただし4秒間が限度。)⇒4秒後に"Main program done."と5秒後に"I'm done"
            Task.WaitAll(new[] { t, t2 }, 4000, token);

            Console.WriteLine($"Task t status is {t.Status}");
            Console.WriteLine($"Task t2 status is {t2.Status}");

            Console.WriteLine("Main program done.");
            Console.ReadKey();
        }
    }
}
```

#### 例外処理

タスク内で例外を投げても、実際のプログラムは正常終了しているように見える。

```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace TPL
{
    class Program
    {
        static void Main()
        {
            var t = Task.Factory.StartNew(() =>
            {
                throw new InvalidOperationException("Can't do this!") { Source = "t" };
            });
            var t2 = Task.Factory.StartNew(() =>
            {
                throw new AccessViolationException("Can't access this!") { Source = "t2" };
            });

            Console.WriteLine("Main program done.");
            Console.ReadKey();
        }
    }
}

```

タスクを Wait すれば、例外を取得できる。(プログラムがクラッシュする。)

```cs
var t = Task.Factory.StartNew(() =>
{
    throw new InvalidOperationException("Can't do this!") { Source = "t" };
});
var t2 = Task.Factory.StartNew(() =>
{
    throw new AccessViolationException("Can't access this!") { Source = "t2" };
});

Task.WaitAll(t, t2);

Console.WriteLine("Main program done.");
Console.ReadKey();
```

Wait した箇所を TryCatch で囲めばプログラムをクラッシュさせずに例外処理できる。

```cs
try
{
    Task.WaitAll(t, t2);
}
catch (AggregateException ae)
{
    foreach (var e in ae.InnerExceptions)
    {
        Console.WriteLine($"Exception {e.GetType()} from {e.Source}");
    }
}
```

AggregateException の Handle メソッドを使うことで、特定の例外のみ処理することができる。

> public void Handle(Func<Exception, bool> predicate);

上記は、Exception を引数に取り、bool 型を返すラムダ式を意味する。

http://techtipshoge.blogspot.com/2014/06/c9actionfuncpredicate.html

```cs
try
{
    Task.WaitAll(t, t2);
}
catch (AggregateException ae)
{
    ae.Handle(e =>
    {
        if (e is InvalidOperationException)
        {
            Console.WriteLine("Invarid op!");
            return true;
        }
        return false;
    });
}
```

### Data Sharing and Synchronization

これ以上分解できないような処理のことを atomic と呼ぶ。

#### critical section

以下の処理を実行してみると、結果が 0 になることは少ないはず。
これは、Deposit と WithDraw の処理が Atomic ではないから。

例えば、Deposit の処理は
1.Balance に amount を足す。
2.Balance をセットする。

という 2 つの処理からなっており、分解不可能ではないので、こういった 0 にならないという現象が発生する。

これらを避けるには、Critical Section をセットする必要がある。

```cs
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

namespace TPL
{
    public class BankAccount
    {
        public int Balance { get; private set; }

        public void Deposit(int amount)
        {
            Balance += amount;
        }

        public void WithDraw(int amount)
        {
            Balance -= amount;
        }
    }
    class Program
    {
        static void Main()
        {
            var tasks = new List<Task>();
            var ba = new BankAccount();

            for (int i = 0; i < 10; i++)
            {
                tasks.Add(Task.Factory.StartNew(() =>
                {
                    for (int i = 0; i < 1000; i++)
                    {
                        ba.WithDraw(100);
                    }
                }));

                tasks.Add(Task.Factory.StartNew(() =>
                {
                    for (int i = 0; i < 1000; i++)
                    {
                        ba.Deposit(100);
                    }
                }));
            }
            Task.WaitAll(tasks.ToArray());

            Console.WriteLine($"Final Balance is {ba.Balance}");
        }

    }
}
```

以下のように lock ステートメントをセットすれば、処理が同期的に行われる。

```cs
public object padlock = new object();
public int Balance { get; private set; }

public void Deposit(int amount)
{
    lock (padlock)
    {
        Balance += amount;
    }
}

public void WithDraw(int amount)
{
    lock (padlock)
    {
        Balance -= amount;
    }
}
```

また、lock ステートメントの代わりに、interlocked クラスの Add メソッドを使っても良い。

```cs
public static class Interlocked
{
    public static int Add(ref int location1, int value);
    public static long Add(ref long location1, long value);
}

```

プロパティを ref に渡すことはできないので、用意する。

Balance プロパティで Alt +Enter で自動プロパティに変換する、を選択。

```cs
public class BankAccount
{
    public object padlock = new object();
    private int balance;

    public int Balance
    {
        get => balance;
        private set => balance = value;
    }

    public void Deposit(int amount)
    {
        Interlocked.Add(ref balance, amount);
    }

    public void WithDraw(int amount)
    {
        Interlocked.Add(ref balance, -amount);
    }
}
```

### Concurrent Collections

TryAdd を用いることで、同一の辞書に大して複数のスレッドから更新を行っている。
スレッドセーフ。

```cs
using System;
using System.Collections.Concurrent;
using System.Threading.Tasks;

namespace TPL
{
    class Program
    {
        private static ConcurrentDictionary<string, string> capitals =
            new ConcurrentDictionary<string, string>();
        private static void AddParis()
        {
            bool success = capitals.TryAdd("France", "Paris");
            string who = Task.CurrentId.HasValue ? ("Task" + Task.CurrentId) : "Main Thread";
            Console.WriteLine($"{who} {(success ? "added" : "did not add")} the element");
        }
        static void Main()
        {
            AddParis();
            Task.Factory.StartNew(() =>
            {
                AddParis();
            }).Wait();
        }
    }
}
```

### Async

以下のように非同期でない Calc メソッドで Thread.Sleep()を使用すると、こちら側からの動作を受け付けなくなる。

```cs
using System;
using System.Threading;
using System.Windows.Forms;

namespace Async
{
    public partial class Form1 : Form
    {
        public int Calc()
        {
            Thread.Sleep(5000);
            return 123;
        }

        public Form1()
        {
            InitializeComponent();
        }

        private void button1_Click(object sender, EventArgs e)
        {
            int n = Calc();
            label1.Text = n.ToString();
        }
    }
}
```

async キーワードを使わなくても、非同期処理を実装することは可能。

```cs
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace Async
{
    public partial class Form1 : Form
    {
        public Task<int> CalcAsync()
        {
            return Task.Factory.StartNew(() =>
            {
                Thread.Sleep(5000);
                return 123;
            });
        }

        public Form1()
        {
            InitializeComponent();
        }

        private void button1_Click(object sender, EventArgs e)
        {
            var calc = CalcAsync();
            calc.ContinueWith(t =>
            {
                label1.Text = t.Result.ToString();
            }, TaskScheduler.FromCurrentSynchronizationContext());
        }
    }
}
```

async、await を使えばもっと簡単に書ける。

```cs
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace Async
{
    public partial class Form1 : Form
    {
        public async Task<int> CalcAsync()
        {
            await Task.Delay(5000);
            return 123;
        }

        public Form1()
        {
            InitializeComponent();
        }

        private async void button1_Click(object sender, EventArgs e)
        {
            var calc = await CalcAsync();
            label1.Text = calc.ToString();
        }
    }
}
```

#### Task.Run

Task クラスの Run メソッドについて。
Task.Factory.StartNew()とほぼ同じ。8 つのオーバーロードを持つ。

- Task か Task<T>か
- キャンセル可能か、不可能か
- 同期か非同期か

例えば、以下のように内部でタスクを返すタスク t を用意する。
この時、戻り値は Task<Task>となる。

```cs
public Task<Task> CalcSomething()
{
    var t = Task.Factory.StartNew(() =>
    {
        Task innerTask = Task.Factory.StartNew(() => { });
        return innerTask;
    });
    return t;
}
```

内部のタスクが戻り値 int を返す場合は、Task<Task<int>>となる。

```cs
public Task<Task<int>> CalcSomething()
{
    var t = Task.Factory.StartNew(() =>
    {
        Task<int> innerTask = Task.Factory.StartNew(() => 123);
        return innerTask;
    });
    return t;
}
```

内部のタスクを async に書き換える。

```cs
var t = Task.Factory.StartNew(async () =>
{
    await Task.Delay(5000);
    return 123;
});
```

UnWrap()を追加する。これで t の型が Task<Task<int>>から Task<int>に変化する。

```cs
var t = Task.Factory.StartNew(async () =>
{
    await Task.Delay(5000);
    return 123;
}).Unwrap();
```

Task.Run はこの Task.Factory.StartNew(...).Unwrap();
のショートカットと考えればよい。

await は UnWrap()と同じ。

以下のソースは全て正しい。
async メソッドは全て await か Unwrap を付ける必要がある。

```cs
int t = await await Task.Factory.StartNew(async () =>
{
    await Task.Delay(5000);
    return 123;
});

int t = await Task.Factory.StartNew(async () =>
{
    await Task.Delay(5000);
    return 123;
}).Unwrap();

Task<int> t = await Task.Factory.StartNew(async () =>
{
    await Task.Delay(5000);
    return 123;
});

Task<int> t = Task.Factory.StartNew(async () =>
{
    await Task.Delay(5000);
    return 123;
}).Unwrap();

Task<Task<int>> t = Task.Factory.StartNew(async () =>
{
    await Task.Delay(5000);
    return 123;
});

Task<int> t = Task.Factory.StartNew(() =>
{
    Task.Delay(5000);
    return 123;
});
```

### コンストラクタで非同期処理を行う方法

方法 1.インスタンス生成後に別途非同期メソッドを呼び出す。⇒ 呼び忘れが心配なので非推奨

```cs
using System;
using System.Collections.Concurrent;
using System.Threading;
using System.Threading.Tasks;

namespace TPL
{
    public class Foo
    {
        public Foo()
        {

        }

        public async Task<Foo> InitAsync()
        {
            await Task.Delay(1000);
            return this;
        }

    }
    class Program
    {
        static async Task Main()
        {
            Foo foo = new Foo();
            await foo.InitAsync();
        }
    }
}
```

方法 2.コンストラクタを private にして、static なファクトリメソッドを用意する。

```cs
using System;
using System.Collections.Concurrent;
using System.Threading;
using System.Threading.Tasks;

namespace TPL
{
    public class Foo
    {
        private Foo()
        {

        }

        private async Task<Foo> InitAsync()
        {
            await Task.Delay(1000);
            return this;
        }

        public static Task<Foo> CreateAsync()
        {
            var result = new Foo();
            return result.InitAsync();
        }

    }
    class Program
    {
        static async Task Main()
        {
            Foo x = await Foo.CreateAsync();
        }
    }
}
```

#### ValueTask<T>

Task<T>の軽量版。リソース割り当てによるオーバーヘッドを減らす。

そもそも Task とは、ある操作が完了することを約束するもの。
ContinueWith()を繋げて書くことで、タスク後の処理を指定できる。

.Net Core 2.0 から利用可能になったのが ValueTask
ValueTask が必ずしも速いわけではない。

### Linq

Linq がないと for 文が必要になる。

```cs
var results = new List<int>();
foreach (int x in input)
{
    int y = x * x;
    if (y > 10) results.Add(y);
}
return results;

return input.Select(x => x * x).Where(y => y > 10).ToList();
```

Linq にはクエリ構文とメソッド構文の 2 種類がある。
メソッド構文が一般的。

クエリ構文は、分かりづらく使っている人も少ない。メソッド構文に比べて優れている点も少ない。

Linq オペレーター(演算子)は高階関数(引数や戻り値に関数を使う関数)
predcicate(T 型の要素を受け取り、bool 値を返す関数)を引数に取る場合が多い。
predicate = Func<T, bool>のこと。

★
Linq 演算子は T の拡張メソッドとして実装されている。

### IEnumerable

IEnumerable<T>は Linq で非常に良く用いられるインターフェース。
.net のコレクションタイプ(list, array, hashtag, stac...)では全てこのインターフェースを実装している。

IEnumerable<T>の T には Func<>型の引数を取る。
これを Linq to Object と呼ぶ。

また、実際のデータベースを操作したい場合には、IQueryable<T>という別のインターフェースを用いる。

IQueryable<T>の T には Expression<>型の引数を取る。
これを Linq to Entities と呼ぶ。

IEnumerable 型を返す method は、イテレータメソッドと呼ばれる。
yield return x;のように書くと、単一の値を返し、
yield break;でイテレーションを終了させる。

### Enumerable

インターフェースではなく実際のクラスとして機能する。
静的なメソッドをいくつか持っている。(Enumerable.Range()など)
Enumerable 型には Linq の拡張メソッドの中身が全て含まれている為、実質的にここが Linq の本体と言える。

### Operator

IEnumerable 型の演算子にはいくつかの種類がある。

Count、ToArray のように即時コレクション全体を呼び出す(invoke)演算子

Select のように、計算時にのみコレクションを呼び出す遅延演算子

### 演習

Takes a sequence of integers
Remove all odd integers, leaving only even ones
Squares each integer
Removes any resulting integer that is greater than 50
For example, given the input sequence from 1 to 10, the output will be { 4, 16, 36 }.

```cs
using System;
using System.Linq;
using System.Collections.Generic;

namespace Coding.Exercise
{
    public class Exercise
    {
        public static IEnumerable<int> MyFilter(IEnumerable<int> input)
        {
            // todo
            return input.Where(n => n % 2 == 0).Select(x => x * x).Where(y => y <= 50);
        }
    }
}
```
