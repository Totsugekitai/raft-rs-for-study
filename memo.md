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

