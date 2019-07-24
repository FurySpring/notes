Git常规操作：

1. ##### git log

   1. 统计个人代码量：

      `git log --author="spring" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add , subs, loc }' -`

   2. 统计贡献值：

      `git log --pretty='%aN' | sort -u | wc -l`

   3. 查看排名前5的贡献者：

      `git log --pretty='%aN' | sort |uniq -c | sort -kl -n -r | head -n 5`

2. ##### git_stats

   1. 