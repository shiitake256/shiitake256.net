---
date: 2025-05-16T23:00:31+09:00
title: Dockerのuserns-remapを有効化したときの備忘録
updated: 2025-05-22T08:20:22
tags:
  - Docker
---

## 概要

Docker の userns-remap を有効化して、既存のコンテナを userns-remap 環境に移行したときの備忘録です。

`userns-remap`（ユーザー名前空間リマップ）は、コンテナ内のユーザーをホストの低権限ユーザーにマッピング（リマップ）する仕組みです。

参考: [ユーザ名前空間でコンテナを分離 — Docker-docs-ja 24.0 ドキュメント](https://docs.docker.jp/engine/security/userns-remap.html)

## 環境

- OS: Fedora Linux 41 (KDE Plasma) x86_64
- Kernel: Linux 6.14.6-200.fc41.x86_64
- Docker: version 27.3.1, build 2.fc41

## 手順

### Docker 設定ファイルに userns-remap を追加

```diff: /etc/docker/daemon.json
 {
     "runtimes": {
         "nvidia": {
             "args": [],
             "path": "nvidia-container-runtime"
         }
-    }
+    },
+    "userns-remap": "dockremap"
 }
```

### Docker デーモンを再起動

デーモン起動時に自動的に`dockremap`ユーザーと`/var/lib/docker/<uid>.<gid>`ディレクトリが自動作成されます。

自動作成させるために一旦 Docker デーモンを起動して、自動作成されたことを確認したら Docker デーモンを停止して移行作業を行います。

```bash
sudo systemctl restart docker.service docker.socket
```

### ディレクトリが作成されたことを確認

/var/lib/docker/_\<uid\>.\<gid\>_ ディレクトリが作成されていることを確認。

```bash
ls -l /var/lib/docker
```

コマンド実行例:

```log
$ sudo ls -l /var/lib/docker/
[sudo] user02 のパスワード:
合計 4
drwx--x---. 1 root dockremap-root   154  5月 17 19:28 100000.100000
drwx--x--x. 1 root root             158  1月 26 02:03 buildkit
drwx--x---. 1 root root            1408  5月 11 02:35 containers
-rw-------. 1 root root              36  1月 26 00:47 engine-id
drwx------. 1 root root              16  1月 26 00:47 image
drwxr-x---. 1 root root              10  1月 26 00:47 network
drwx--x---. 1 root root           25954  5月 11 02:35 overlay2
drwx------. 1 root root              20  1月 26 00:47 plugins
drwx------. 1 root root               0  5月 10 12:18 runtimes
drwx------. 1 root root               0  1月 26 00:47 swarm
drwx------. 1 root root              50  5月 11 01:06 tmp
drwx-----x. 1 root root             398  5月 10 12:18 volumes
$
```

### ユーザーが作成されたことを確認

```bash
id dockremap
```

コマンド実行例:

```log
$ id dockremap
uid=975(dockremap) gid=972(dockremap) groups=972(dockremap)
$
```

### Docker デーモンを停止

```bash
sudo systemctl stop docker.service docker.socket
```

### データの移行

root 用のディレクトリから userns-remap 用のディレクトリにデータを移行します。

```bash
rsync -aHAX /var/lib/docker/buildkit /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/containers /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/engine-id /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/image /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/network /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/overlay2 /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/plugins /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/runtimes /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/swarm /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/tmp /var/lib/docker/100000.100000/
rsync -aHAX /var/lib/docker/volumes /var/lib/docker/100000.100000/
```

### オーナーの変更

```bash
sudo -i
shopt -s extglob

chown root:100000 /var/lib/docker/100000.100000
chown root:100000 /var/lib/docker/100000.100000/overlay2/!(l)
chown -R 100000:100000 /var/lib/docker/100000.100000/overlay2/*/diff
chown root:100000 /var/lib/docker/100000.100000/containers/*
chown 100000:100000 /var/lib/docker/100000.100000/containers/*/hostname
chown 100000:100000 /var/lib/docker/100000.100000/containers/*/resolv.conf
chown root:root /var/lib/docker/100000.100000/containers/*/resolv.conf.hash
chown root:100000 /var/lib/docker/100000.100000/containers/*/mounts
chown root:100000 /var/lib/docker/100000.100000/containers/*/checkpoints
chown -R 100000:100000 /var/lib/docker/100000.100000/volumes/*/_data
```

### SELinux コンテキストの追加

userns-remap 機能を SELinux のファイルコンテキストを追加。

```bash
sudo semanage fcontext -a -t container_ro_file_t '/var/lib/docker/[0-9]+\.[0-9]+/overlay2(/.*)?'
```

### SELinux ラベルの再設定

```bash
sudo restorecon -r /var/lib/docker/100000.100000
```

### Docker デーモンを起動

```bash
sudo systemctl stop docker.service docker.socket
```
