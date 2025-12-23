# bigdata_kiban

Hadoop (BigTop) + Hive (PostgreSQL metastore) を Ubuntu 24.04 限定で構築する Ansible 一式です。

## Ubuntu 24.04 限定
site.yml と各 role の先頭で `assert` を行い、Ubuntu 24.04 以外では必ず失敗します。

## 事前準備
```bash
git clone git@github.com:naritomo08/github_kiban.git
cd github_kiban
ansible-galaxy collection install community.postgresql
```

## 実行
```bash
ansible-playbook -i inventory.ini site.yml
```

## 変数
`group_vars/all.yml` を環境に合わせて調整してください（IP/ホスト名、BigTop repo URL、メモリ等）。
