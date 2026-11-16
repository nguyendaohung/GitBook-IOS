# 😁 GIT

**Merge và rebase** là hai phương pháp khác nhau để tích hợp các thay đổi từ các nhánh khác nhau trong quản lý mã nguồn.

**Merge (kết hợp)** là quá trình thêm các thay đổi từ một nhánh vào nhánh khác. Với merge, các thay đổi từ nhánh nguồn được giữ nguyên và thêm vào nhánh đích có thể bằng fast-forward hoặc một merge commit mới. Điều này có nghĩa là lịch sử của các nhánh được giữ nguyên và hiển thị rõ ràng trên đồ thị git.

**Rebase (sắp xếp lại)** là quá trình di chuyển các thay đổi từ một nhánh và đặt chúng lên đỉnh của một nhánh khác. Với rebase, các thay đổi từ nhánh nguồn được đặt lên đỉnh của nhánh đích dưới dạng một chuỗi commit mới. Điều này có nghĩa là lịch sử của các nhánh được sắp xếp lại một cách tuần tự và như vậy, sẽ không có commit gộp riêng biệt như trong trường hợp của merge. Rebase sẽ move base lên commit mới nhất của nhánh cha

Vì các thay đổi được sắp xếp lại trong rebase, việc giải quyết xung đột có thể khó hơn so với merge, vì các thay đổi được áp dụng theo thứ tự và không được tách ra như trong trường hợp của merge.

**Sử dụng Merge khi:**

* Cần tích hợp các thay đổi từ nhiều người dùng hoặc nhánh trong một thời điểm cụ thể.
* Các nhánh cần được giữ nguyên lịch sử riêng biệt.
* Có xung đột giữa các commit và cần giải quyết thủ công hoặc sử dụng công cụ hỗ trợ để giải quyết.
* Không cần xóa các commit không cần thiết hoặc lịch sử không cần thiết.

**Sử dụng Rebase khi:**

* Cần tạo một lịch sử commit liền mạch và không bị gián đoạn.
* Muốn giữ các commit của nhánh nguồn nhưng muốn chúng được đặt lên đỉnh của nhánh đích.
* Không có xung đột hoặc ít xung đột giữa các commit.
* Cần replay commit trên base mới; interactive rebase mới có thể squash/drop theo chủ đích.

## Git thực tế và Q&A

```bash
git status
git switch -c feature/catalog
git add .
git commit -m "feat: add catalog search"
git fetch origin
git rebase origin/main
# Resolve conflict, git add file, git rebase --continue
git push -u origin feature/catalog
```

Rebase rewrite commit IDs, không xóa commit tùy ý nếu không interactive/squash/drop. Không rebase shared branch mà không thống nhất team; force-with-lease chỉ khi cần và được phép, tránh force push main. Merge có thể fast-forward (không merge commit) hoặc tạo merge commit; preserve history. Squash merge gộp changes thành commit mới. revert tạo commit đảo changes, reset di chuyển ref/index/worktree theo mode; reset --hard có thể mất local work. Cherry-pick copy commit patch, conflict vẫn cần review.

**Q: merge vs rebase?** Merge kết hợp history, rebase replay commit trên base mới và rewrite IDs.

**Q: revert vs reset?** Revert an toàn cho shared history vì commit mới; reset thay local ref, có thể mất work.

**Q: Conflict xử lý?** Đọc cả hai changes, resolve theo behavior, build/test, stage và continue/commit.

**Q: GitBook sync nên sửa main trực tiếp?** Ưu tiên branch/PR review, giữ SUMMARY/links/assets, merge theo sync config; không rewrite main.

**Q: Pull request vs commit?** Commit snapshot change; PR review/merge branch, có diff/checks.
