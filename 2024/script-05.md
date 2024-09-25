ではどういった場合にmonorepoにするべきなのでしょうか?

まず前提として、ひとつ目に、関係あるもの同士を同じ個体にまとめるのは管理コストを下げることに繋がります。
これは、先ほど図でも示しましたが、具体的な例として、個別のrepositoryでのClone, PR作成などを行う必要がなくなるからです。

次に、個体を分割することは管理コストを上げることになります。
これは先ほどと真逆のことを言っているので異論はないかなと思いますが、複数に散らばれば当然別々での作業が必要になりますし、それはすなわち管理コストが上がっていることを表しています。

3つ目の大前提としては、管理可能な個体の規模には限界があります。
これも先ほどのメリットデメリットでの話にも出てきた通り、あまりにも多くのものを単一の個体として扱うとパフォーマンスやそもそも物理的な制約などに引っかかるためです。

そしてここから自分たちは、3番目に引っかかるような上限を超えない限りはmonorepoが良いのではないか...?と考えました。


ではどのような閾値を持ってmonorepoを分割するべきなのでしょうか?
ここでひとつ例を考えてみましょう。
とあるそこそこ大きな会社のprojectがあるとして、これら全てを単一のrepositoryに入れたら何が起こるでしょうか?

まず一番初めに出てくるのは、
1. 人間による制約です。
例えば、PRの処理が早すぎて追いつかなかったり、
そのrepositoryに含まれる情報が多すぎて整理ができなかったりすると思います。
これではパフォーマンスが帰って落ちてしまうと考えられるため分割するべきですよね。

2. 次は人間以外による制約です。
具体的にはrepositoryが重すぎでcloneするのに時間がかかりすぎたり、重すぎてそもそも開くことができなかったり、
あとは部分的にcloneしたり(最近は対応されているツール多いですが)特定のfolderだけど検知してCI回したり、pre-commitでも同様のことをしたくても行えなかったりします。
そしてもしかしたらgit自体が落ちてしまうケースもあり得るかもしれません。
こういう場合にもそもそも単一のrepositoryで管理するべきでないことがわかります。

3. 最後は人工的な制約で
会社とかだと法律監査要件などで例えばmonorepoで管理していたproductの一つを売却したり、そもそも受託案件とかの場合は同一repoにできないというケースはあると思います。
この場合にはrepositoryは分割するしかなさそうですね。

以上、これら3つの制約に当てはまるような場合にはrepoは分割するべきであると考えられそうです。


では次は具体的にどのように分ける方法があるのかについて考えてみましょう。
また具体的な例を出しますが、このような状況を想像してみてください。
~~~
考えやすくするために(極端な例ですが顔も名前も知らないとします)

ではこのような場合にはどのように分割するのが要さそうでしょうか?


では解説の方ヴァレリさんお願いします。