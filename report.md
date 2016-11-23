#課題 (Cbench のボトルネック調査) レポート

##課題内容

Ruby のプロファイラで Cbench のボトルネックを解析しよう。

以下に挙げた Ruby のプロファイラのどれかを使い、Cbench や Trema のボトルネック部分を発見し遅い理由を解説してください。

### Ruby 向けのプロファイラ

* [profile](https://docs.ruby-lang.org/ja/2.1.0/library/profile.html)
* [stackprof](https://github.com/tmm1/stackprof)
* [ruby-prof](https://github.com/ruby-prof/ruby-prof)

これ以外にもいろいろあるので、好きな物を使ってかまいません。

##方針

今回はプロファイラとして ruby-prof を用いる。ruby-prof で cbench コントローラのボトルネックを発見するために、ターミナルで以下のコマンドを実行した。なお、計測結果を result.txt に出力するようにしている。

```
$ ruby-prof ./bin/trema run ./lib/cbench.rb > result.txt
```

この後すぐに別ターミナルで cbench プロセスを起動し、ベンチマークが終了した後すぐに cbench コントローラのプロセスを終了した。

##実行結果と考察

まず、cbench プロセスによるベンチマークの結果を掲載する。

```
cbench: controller benchmarking tool
   running in mode 'throughput'
   connecting to controller at localhost:6653 
   faking 1 switches :: 10 tests each; 10000 ms per test
   with 100000 unique source MACs per switch
   starting test with 1000 ms delay after features_reply
   ignoring first 1 "warmup" and last 0 "cooldown" loops
   debugging info is off
1   switches: fmods/sec:  101   total = 0.010036 per ms 
1   switches: fmods/sec:  72   total = 0.007167 per ms 
1   switches: fmods/sec:  61   total = 0.006081 per ms 
1   switches: fmods/sec:  45   total = 0.004463 per ms 
1   switches: fmods/sec:  61   total = 0.006073 per ms 
1   switches: fmods/sec:  56   total = 0.005576 per ms 
1   switches: fmods/sec:  47   total = 0.004662 per ms 
1   switches: fmods/sec:  41   total = 0.004092 per ms 
1   switches: fmods/sec:  37   total = 0.003652 per ms 
1   switches: fmods/sec:  34   total = 0.003348 per ms 
RESULT: 1 switches 9 tests min/max/avg/stdev = 3.35/7.17/5.01/1.21 responses/s
```

次に、 ruby-prof による cbench コントローラの解析結果の一部を以下に掲載する (全結果は[こちら](https://raw.githubusercontent.com/handai-trema/cbench-Takuya-Saitoh/master/result.txt))。

```
* indicates recursively called methods
Measure Mode: wall_time
Thread ID: 20655040
Fiber ID: 20654880
Total: 103.414690
Sort by: self_time

 %self      total      self      wait     child     calls  name
  3.08     13.376     3.187     0.000    10.189   382754  *BinData::BasePrimitive#snapshot
  2.97     19.022     3.076     0.000    15.946   257187   BinData::Struct#instantiate_obj_at
  2.67     10.338     2.764     0.000     7.574   370992  *BinData::BasePrimitive#_value
  2.19      2.260     2.260     0.000     0.000   160038   String#sub
  2.12     19.001     2.190     0.000    16.811   160030   BinData::Struct#find_obj_for_name
  1.95      7.215     2.021     0.000     5.194   160030   BinData::Struct#base_field_name
  1.94      2.006     2.006     0.000     0.000   160030   Array#index
  1.92     18.354     1.984     0.000    16.370   139547   BinData::Base#new
  1.87      1.931     1.931     0.000     0.000   160030   String#to_sym
  1.81      5.638     1.869     0.000     3.769   102845   Kernel#dup
  1.72      2.278     1.780     0.000     0.498   152220   BinData::Struct::Snapshot#[]=
  1.61      1.665     1.665     0.000     0.000   147345   Kernel#initialize_copy
  1.44      1.488     1.488     0.000     0.000   310315   Symbol#to_s
  1.39      1.434     1.434     0.000     0.000   289070   BasicObject#!
  1.34     11.331     1.381     0.000     9.949   120619   BinData::Primitive#method_missing
  1.31      1.445     1.353     0.000     0.092   230771   BinData::Base#get_parameter
  1.26      1.301     1.301     0.000     0.000   115608   BinData::BasePrimitive#initialize_instance
  1.17      4.086     1.208     0.000     2.878   142931  *BinData::BasePrimitive#method_missing
  1.06      1.093     1.093     0.000     0.000   187642   Symbol#to_sym
```

一番左の項目である %self は、プロセス全体の中で name のメソッドの処理に費やされた時間の割合を表している。つまりこの値が大きいメソッドは、プロセスにおけるボトルネックになっていると言えるのである。また、 ruby-prof のこの計測結果は、 %self が大きい順にデータを表示していて、上記の %self が大きいメソッドを見てみると、その多くが BinData 関連のメソッドであることがわかる。 BinData とは ruby においてバイナリデータを扱う際に使用される gem である。ここで、 cbench コントローラは Packet In を受け取ると Flow Mod を返すという単純な動作しかしないところから、これら BinData 関連のメソッドは、 cbench コントローラが受け取った Packet In のバイナリデータを読み取ったり、生成した Flow Mod を cbench プロセスに送信する際にバイナリデータに変換する時に呼び出されていると推測される。したがって、これらバイナリデータを扱う処理が cbench コントローラのボトルネックになっていると考えられる。
