# クラスから離れよう｜Unity・C#の速度のコツ
## 第ゼロ部｜前置き
ごきげんよう。サイドでございます。これは私の初めての記事なので、記事の書き方のコツはまだ分からず、これから時間をかけて勉強していきたいと思います。そして日本語はまだ勉強中ですので、忌憚のない評価をいただけると嬉しいです。
## 第一部｜前置き
**この記事は上級者向けのため、初級者向けの話題は省略します。🙇**

実際のプロジェクトでは、**パフォーマンスと使いやすさのトレードオフ**が常に問題になります。しかしこの記事では、その複雑な問題を一旦脇に置き、パフォーマンスのみを追求します。
これから、Unityにおける様々なクラスに依存した実装を分析し、どのようにクラスから離れ、そして可能な限り使いやすさを保ちながら書き直します。  
一言で、この記事の主張は「一旦、クラス＝悪と想定しよう」です。

## 第二部｜コンポジション
継承を用いて様々な実装を再利用することは多いですね。下記の例を見てください。
```cs
public abstract class Entity
{
    public abstract bool CanTakeDamage();
    public abstract bool CanAttack();
    public abstract State GetState();
    public abstract bool CanMove();
    public abstract bool CanDie();
}
public abstract class Human : Entity
{
    public override bool CanTakeDamage() => true;
    public override bool CanMove() => true;
    public override bool CanDie() => true;
}
public class Peasant : Human
{
    public override bool CanAttack() => false;
    public override State GetState() => ...;
}
public class Soldier : Human
{
    public override bool CanAttack() => true;
    public override State GetState() => ...;
}
```
では、このような継承の使い方を、以下のようにコンポジションを用いてクラス無しで書き直せます。
```cs
public interface ICanTakeDamage { }
public interface ICanAttack { }
public interface IHasState 
{
    public State GetEntityState();
}
public interface ICanMove { }
public interface ICanDie { }

public interface IHuman : ICanTakeDamage, ICanMove, ICanDie { }
public struct Peasant : IHuman, IHasState
{
    public State GetEntityState() => ...;
}
public struct Soldier : IHuman, IHasState, ICanAttack
{
    public State GetEntityState() => ...;
}
```
このように、`interface`をコンポーネントとして扱い、`struct`を作成します。この後、もし「人間だけど人間らしくない」人物が出てきても、簡単に適切な`interface`を付けて作成できます。しかし、前の継承の例ではそのような人物を作成したい場合、親クラスそのものを修正しなければならない状況になります。さらに、`class`型が不要になったため、`struct`型に変更しました。

> このようなコンポジションでは、もちろんボクシングを避けることが重要です。ジェネリック制約によって回避する方法は既知と想定しますが、念のため短い例を挙げましょう。
> ```cs
> public static bool DoStatesMatch<T>(in T entity, State state) where T : IHasState
> {
>     // ボクシング無しでentityを利用可能
>     return entity.GetEntityState() == state;
> }
> ```
> 一方、ボクシングを回避できない場合もあります。（例えば`if (entity is IHasState state)`のような状況）実際のプロジェクトの場合によって`interface`が不要で、`enum`などを採用すれば良い状況もあります。
## 第三部｜委譲
下記の継承を用いた例を見てください。
```cs
public class Timer
{
    public void Reset() { ... }
    public void Start() { ... }
    public void Stop() { ... }
    public long GetTick() { ... }
}

public class LocalizedTimer : Timer
{
    public string GetLocalizedTime() 
    {
        var ticks = GetTick();
        ...
    }
}
```
継承無しのバージョンは以下の通りです。
```cs
public struct Timer
{
    public void Reset() { ... }
    public void Start() { ... }
    public void Stop() { ... }
    public long GetTick() { ... }
}
public struct LocalizedTimer
{
    public Timer timer;
    public string GetLocalizedTime()
    {
        var ticks = timer.GetTick();
        ...
    }
}
```
文字通り委譲していますね。継承機能ではなく、簡単にメンバーに委譲して`tick`を入手します。もし必要であれば、`timer`をプライベートにし、`Timer`の全ての関数を`LocalizedTimer`内にも書き、関数ごとにプライベートな`_timer`へ委譲することもできます。
## 第四部｜拡張メソッド
第三部と同じケースでは、もう一つの方法もあります。それは拡張メソッドという機能です。拡張メソッドは多くのプログラミング言語では珍しい機能ですが、幸い現代のUnityのC#では使用可能です。では第三部の例を：
```cs
public struct Timer
{
    public void Reset() { ... }
    public void Start() { ... }
    public void Stop() { ... }
    public long GetTick() { ... }
}
public static class TimerExtensions
{
    public static string GetLocalizedTime(this Timer timer)
    {
        var ticks = timer.GetTick();
        ...
    }
}
```
拡張メソッドではメンバー変数を追加できないため、委譲による実装を全て無理に拡張メソッドで置き換えることはできません。
## 第オマケ部｜データ指向設定
これまでの継承による設計を分析し、継承無しで書き直したことで、部分的に`class`を使わずにシステムを作成できることが明らかになりましたね。もしある部分のデータが全て`struct`であれば、その部分のメモリレイアウトは容易に想像できます。すなわち、完璧に連続したデータです。完璧であるからこそ、様々な特別な機能も使用可能になります。その一つが`stackalloc`です。
```cs
public static void BenchmarkCommandDuration(string cmd)
{
    Span<Timer> timers = stackalloc Timer[100];
    long total = 0;
    for (int i = 0; i < 100; i++)
    {
        timers[i].Reset();
        timers[i].Start();
        ExecuteInternal(cmd);
        timers[i].Stop();
        total += timers[i].GetTick();
    }
    // バブルソート
    for (int i = 0; i < 100 - 1; i++)
        if (timers[i].GetTick() > timers[i + 1].GetTick())
            (timers[i], timers[i + 1]) = (timers[i + 1], timers[i]);
    
    var medianTime = timers[100 / 2];
    Debug.LogFormat("executed \"{0}\" | med: {1} avg: {2}", 
        cmd, 
        medianTime.GetLocalizedTime(), 
        new Timer(total / 100).GetLocalizedTime());
}
```
ヒープメモリを使わずに配列を使えるようになります。この例では`timers`配列はスタック上に確保されますので、ヒープ割り当てや解放が不要になり、高速化のテクニックとなります。

メリットはスタック利用だけに限らず、**データの保存・読み込み、そしてランタイムでのデータアクセス**も楽になります。

AAAゲーム業界ではデータ指向設計を採用することが多いです。UnityのDOTSのECSもそのようなシステムです。高速であるため、ゲームにとって非常に有用な指向設計だと考えられています。
> データ指向設計が気に入ったら、**キャッシュラインの最適化**について調べてみてください。
# 第ことわざ部
> **急がば回れ**

速く決めた時こそ、もう一度ゆっくり考えてください。