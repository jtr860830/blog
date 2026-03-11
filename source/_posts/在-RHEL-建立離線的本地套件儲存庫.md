---
title: 在 RHEL 建立離線的本地套件儲存庫
date: 2026-03-10 00:00:00
categories:
  - Linux
tags:
  - Linux
---

[![hackmd-github-sync-badge](https://hackmd.io/YVUCxMqESeaZ5pgqRmQJww/badge)](https://hackmd.io/YVUCxMqESeaZ5pgqRmQJww)

有鑑於最近需要在 Air-Gap 的環境安裝 Kubernetes (RKE2)，所以將此方法記錄下來，主要是因 RKE2 所需要的系統套件需要透過 rpm 去安裝。會想要用這個方式的原因是因為筆者覺得比較保險，因為 `dnf` 會幫忙解析套件的相依性，比起直接使用 `rpm` 指令安全穩定的多。

# 環境

需在可上網的環境建立一版本相同的 RHEL，像如果 Air-Gap 的環境是 RHEL 9.6，那就需要準備一台可連網的 RHEL 9.6 server，去製作套件庫

# 步驟

先安裝好可連網的 RHEL server，並確定套件管理器 `dnf` 可以正常運作，這邊要演示的是要安裝 `rke2-selinux` 這個 rpm，因為安裝套件不是單單下載這個 rpm 就好，還需要包含此套件的相依性套件

## 匯入需要的第三方套件儲存庫

1. 使用 `vim add-repo.sh` 編輯檔案，內容如下:

```
export RKE2_MINOR=35
export LINUX_MAJOR=9 # or 10 etc
cat << EOF > /etc/yum.repos.d/rancher-rke2-1-${RKE2_MINOR}-latest.repo
[rancher-rke2-common-latest]
name=Rancher RKE2 Common Latest
baseurl=https://rpm.rancher.io/rke2/latest/common/centos/${LINUX_MAJOR}/noarch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key

[rancher-rke2-1-${RKE2_MINOR}-latest]
name=Rancher RKE2 1.${RKE2_MINOR} Latest
baseurl=https://rpm.rancher.io/rke2/latest/1.${RKE2_MINOR}/centos/${LINUX_MAJOR}/x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://rpm.rancher.io/public.key
EOF
```

2. 執行 `sudo bash add-repo.sh`
3. 更新套件清單 `sudo dnf update -y`

> 這邊若是已有更新的 RHEL 9，可以使用這個指令 `sudo subscription-manager release --set=9.6` 鎖定套件只會更新在 9.6，不會更新到 9.7 或更新的版本

## 下載所需的 rpm 檔案，並製作出儲存庫

1. 建立資料夾 `mkdir rpm-repo`
2. 下載需要的工具 `sudo dnf install -y createrepo_c dnf-plugins-core`
3. 下載需要的 rpm `dnf download --resolve --destdir=./rpm-repo rke2-selinux`
> 如果需要完整的 rpm 套件檔案，可以加上 `--alldeps` 但就會連系統預裝的套件都下載下來，有可能導致相依性衝突
> 可使用 `dnf repoquery --requires <套件名稱>` 檢查第一層的相依性套件
> 若遠端的儲存庫套件不多也可以考慮直接把所有最新的 rpm 都下載下來，以上方加入的儲存庫為例 `dnf reposync --repo rancher-rke2-common-latest -n`
4. 製作套件儲存庫 `createrepo_c ./rpm-repo/`
5. 打包套件儲存庫 `tar -cvzf rpm-repo.tar.gz rpm-repo`

## 到 Air-Gap 環境匯入套件庫並安裝

1. 將打包玩的 tar 檔案放到 Air-Gap 的機器中
2. 建立存放儲存庫的資料夾 `sudo mkdir -p /opt/rke2/`
3. 解壓縮檔案 `sudo tar -xvzf rpm-repo.tar.gz -C /opt/rke2/`
4. 建立儲存庫設定檔 `sudo vim /etc/yum.repos.d/local.repo`
```
[local-rke2]
name=RKE2 Local Offline Repo
baseurl=file:///opt/rke2/rpm-repo/
enabled=1
gpgcheck=0
```
5. 就能夠安裝需要的套件 `sudo dnf install rke2-selinux`
