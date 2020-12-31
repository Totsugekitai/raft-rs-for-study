# The Raft Consensus Algorithm paper memo

raftを実装する上での論文メモです。

論文：[In Search of an Understandable Consensus Algorithm(Extended Version)](https://raft.github.io/raft.pdf)

## Abstract

- 複製ログを管理するための分散合意アルゴリズム
- わかりやすさを重視
  - 以下の要素を分解して設計
    - リーダー選出
    - ログ複製
    - 安全性

## Introduction

- Paxosは有名な分散合意アルゴリズム
  - 難しい
  - 実際のプロダクトに組み込むのも大変
- Paxosの欠点を克服するためのRaft
  - "強い"リーダーシップ
    - ex. ログエントリはリーダーからのみ他のノードに流れる
  - リーダー選出
    - ランダムなタイマーを用いてリーダー選出
      - ハートビートに対して少しの改良を加えるだけでよい
  - メンバーシップの変更
    - 2つの異なる構成の大部分が移行中に重複する新しいコンセンサスのアプローチをとる

## Replicated state machines

- Replicated state machines は replicated log を用いて実装されることが多い
- Figure 1の例を見る
  - 各サーバはlogをストアする
    - logにはcommand(おそらく命令のこと)列が格納されている
      - 各サーバの各logは同じcommand列が同じ順番で格納される
        - これにより各サーバは同じcommandを同じ順番で実行する
  - state machinesは決定的であるため、各サーバの計算は同じ状態であるし、同じ出力をする
- replicated logの一貫性を保つのがコンセンサスアルゴリズム
  - サーバ内のconsensus moduleがクライアントからのcommandを受け取ってlogに追加する
  - consensus module同士が通信して、サーバがダウンしてもlogの一貫性を保つ必要がある
  - commandが複製された後に、各サーバのstate machineがlogに格納されている順番に計算を行い、クライアントに出力を返す
- ビザンチン状態(結託して過半数が嘘をつくような状態)ではない条件下で、safetyが保たれる
  - 以下の障害に耐性を持つ
    - ネットワーク遅延
    - 分断
    - パケットロス
    - 重複
    - 命令の再実行
- 過半数が操作可能で、クライアントやサーバ間で通信可能である条件下で、functionalである
- logの一貫性を保つ上で、タイミングに依存しない設計
  - clockが壊れたり遅延が大きすぎても、最悪でもavailability problemに抑え込める
- 過半数のサーバがcommandを実行した時点で、クライアントにレスポンスをするようになっている
  - 少数の遅いサーバの影響を受けない

## What's wrong with Paxos?

一旦省略(必要が出たらまとめます)

## Designing for understandability

一旦省略(必要が出たらまとめます)

## The Raft consensus algorithm

- まず識別可能なleaderを選出する
  - leaderは複製ログを管理する完全な責任を与えられる
- leaderはクライアントから来たログエントリを承認し、各サーバに複製し、各サーバのstate machineにログエントリを適用する
  - leaderは各サーバと相談を必要とせずにログエントリをlog内に格納できる
  - データフローはleaderから各サーバのみ
- leaderに障害または疎通が取れなくなったら、新しいleaderの選出が始まる

### Raft basics

- 各ノードのstateはleader,follower,candidate
  - leaderは1ノード、followerはleader以外のノード
- followerはleaderからの応答のみを行う
- candidateはelection時に全ノードがなる状態

- 時間をtermという区切りで管理する
- 各termはelectionから始まる
- electionで新しいleaderが選出されたらnormal operationが続く
- leaderに障害が起きたら次のtermに遷移する
- 各ノードは別ノードのterm値を監視して、遷移があったら自身のterm値を更新する

- Raftを実現するRPCは2つ
  - RequestVote RPC
    - candidateが発するRPC
  - AppendEntries RPC
    - leaderが発し、followerが返答することでハートビートの役割を兼任する

### Leader election

- 各ノードのスタートアップ時は、followerから始まる
- leaderかcandidateから有効なRPCを受け取るまで、followerを維持する
- leaderは定期的にハートビートを送り、権限管理する
  - log entriesが無いAppendEntries RPCをハートビートとする
- 一定期間ハートビートを受け取らなかった際(election timeoutという)、followerはleaderがいないと仮定し、electionを開始する

- electionを開始する際、以下のことを行う
  - current termをインクリメント
  - candidateに遷移
- 自身にvoteするのと他ノードにRequestVote RPCを送るのを並列に行う
- candidateは以下のうち1つが起こるまで、candidateの状態を継続する
  - 自身がelectionに勝つ
  - 他ノードがleaderとして確立したことが確認される
  - 一定時間leaderが現れなかった

- 1つのcandidateに対してクラスタ中の過半数のノードから投票が同一term内であった場合、electionに勝ったとする
- 同一term内において、各サーバは多くても1つのノードに対してvoteを行う
  - 最初に来たものに最初に(voteを)返す方式
- leaderになったノードは、ハートビートを他ノードに送り、権限管理とelectionの抑制を行う

- voteを待っている間、candidateは自分がleaderだと主張するノードからAppendEntried RPCを受け取る可能性がある
  - そのleaderのterm値がcandidateのterm値以上の場合は、そのノードを正当なleaderだと認識し、自身の状態をfollowerに遷移させる
  - そのleaderのterm値がcandidateのterm値未満の場合は、RPCを否認しcandidate状態を継続する

- voteに勝ちも負けもせず、一定時間が経過したとき
  - 新たにelectionを開始する
    - candidateはterm値をインクリメントして新たにRequestVote RPCを別ノードに送る

- election timeoutにランダムな値を用いる
  - ある一定区間の時間からランダムで時間を決定する(e.g. 150-300ms)
  - 多くの場合1つのノードのみがtime outする
    - ノードがelectionに勝ち、他ノードがtime outするまでにハートビートを送り始める

### Log replication

- leaderが選ばれたら、クライアントのリクエストを受け付けるようになる
- 各クライアントのリクエストは各ノードのreplicated state machineが実行するcommandが含まれている
- leaderは新たなエントリとしてlogに対してcommandを付け足し、各ノードに複製するためにAppendEntries RPCを送る
- 安全に複製された場合、leaderはエントリをstate machineに適用し、結果をクライアントに返却する
- followerがクラッシュしたり、実行が遅かったり、パケットがロストしたりした場合は、leaderはAppendEntries RPCを無制限に再送する
  - クライアントにレスポンスした後でも、すべてのfollowerが全てのlog entryを保存するまで繰り返される

- logはstate machine commandがterm値とともに書き込まれる
  - term値はleaderからエントリを受け取った際のもの
- term値はlogと図3のプロパティの不一貫性の検査に用いられる
- 各logエントリは整数のインデックスを持ち、log中の位置を表す

- leaderはいつlogをstate machineに適用するのが安全か決定する
  - (安全だと判断された？)エントリは `committed` と呼ばれる
- Raftはcommitedなエントリは耐久性を持ち、全ての利用可能なstate machineにおいて順番に実行されることが保証される
- 生成されたエントリが過半数のノードで複製されるとleaderによりcommitされる
  - leaderのlogにおいてcommitされたエントリよりも先行するlogエントリもcommitされる
  - 前のleaderによってcommitされたエントリも含む
- ココらへんの微妙なルールや安全性についての議論は5.4節で説明する
- leaderはcommitされたエントリのインデックスは最も高いものを保持する
  - これは未来にAppendEntries RPC(ハートビート含む)で送られるものも含む
  - 他ノードも順序が理解できるようになっている
- あるlogエントリがcommitされたものだとfollowerが知ると、ローカルのstate machineに適用する

- 違ったlog中で同じインデックスと同じtermを持った2つのentryができてしまった場合
  - 同じcommandをストアする
    - leaderは最高でも1termで与えられたlogインデックスにつき1つのエントリしか生成しない
    - logエントリはlog中の位置を変えることはない
  - logは全ての先行するエントリに対して識別可能である
    - AppendEntries RPCにより保証される
      - AppendEntries RPCが送られる際、leaderはlogのインデックスとエントリのtermを含ませる
        - これは新たなエントリの直後に先行するlogである
- followerがlog中に同じインデックスとtermを見つけられなかった際は、新しいエントリを否認する

- 通常のオペレーション中は、leaderとfollowerのlogは一貫性が保たれる
  - この際のAppendEntriesの一貫性チェックは失敗することはない
- leaderがクラッシュした際は、一貫性が保てなくなる
  - 古いleaderは全てのエントリの複製を持っていないかもしれない
  - leaderのみならずfollowerのクラッシュも招く可能性あり

- Raftはこれらの不一貫性を、followerのlogに対してleaderの複製をもたせることで解決している
  - followerのlogでコンフリクトが起きた場合には、leaderのlogを上書きする

- followerのlogが一貫性を保つためにleaderは以下を満たさなければならない
  - 2つのlogが合意する最も最近のlogエントリを見つける
  - そのポイントより後のlogエントリを削除する
  - そのポイントより後のleaderのlogエントリをfollowerに送る
- これらはAppendEntries RPCの一貫性チェックで行われる
- leaderは `nextIndex` を各followerに対して整備する必要がある
  - これはleaderがfollowerに対して送る次のlogエントリのインデックス
- leaderが初めて実権を握った際には、全てのnextIndexの値をlog中の最後の一つのエントリのインデックス+1したものに初期化する
- followerのlogがleaderと一貫していない場合、AppendEntriesの一貫性チェックは次のAppendEntries RPCで失敗する
- 拒否された後は、leaderはnextIndexの値をデクリメントし、AppendEntries RPCを再送する
  - これを繰り返すことで、leaderとfollowerのlogが一貫するポイントまで戻ることになる
