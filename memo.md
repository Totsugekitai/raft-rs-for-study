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

- 
