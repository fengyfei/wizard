# git 命令执行流程

## 入口函数

对于 C 语言，程序入口函数一定是 main，我们首先找到 main 方法:

```C
int main(int argc, const char **argv)
{
	/*
	 * Always open file descriptors 0/1/2 to avoid clobbering files
	 * in die().  It also avoids messing up when the pipes are dup'ed
	 * onto stdin/stdout/stderr in the child processes we spawn.
	 */
	sanitize_stdfds();

	git_setup_gettext();

	attr_start();

	git_extract_argv0_path(argv[0]);

	restore_sigpipe_to_default();

	return cmd_main(argc, argv);
}
```

git 没有常驻后台的服务存在，因此每一条 git 命令，都需要执行 main 中方法，所以不难看出，实际执行命令的方法为 cmd_main。

- cmd_main：

```C
int cmd_main(int argc, const char **argv)
{
	// 如果是 git- 开头的命令
	if (skip_prefix(cmd, "git-", &cmd)) {
		argv[0] = cmd;
		handle_builtin(argc, argv);
		die("cannot handle %s as a builtin", cmd);
	}

	// 跳过执行路径
	argv++;
	argc--;

	// 处理参数
	handle_options(&argv, &argc, NULL);

	// 获取命令
	cmd = argv[0];

	// 设置执行路径
	setup_path();

	while (1) {
		// 实际执行命令
		int was_alias = run_argv(&argc, &argv);
	}

	return 1;
}
```

进入 run_argv 可以看到，执行部分位于 handle_builtin 中。

handle_builtin 中关键代码为：

```C
	if (p->option & RUN_SETUP)
		prefix = setup_git_directory();

	if (!help && p->option & NEED_WORK_TREE)
		setup_work_tree();

	// 根据 cmd 找到对应执行函数
	builtin = get_builtin(cmd);
	if (builtin)
		exit(run_builtin(builtin, argc, argv));
```

get_buildin 根据 cmd 查找执行代码，全部 cmd 映射如下

- 命令列表

```C
static struct cmd_struct commands[] = {
	{ "add", cmd_add, RUN_SETUP | NEED_WORK_TREE },
	{ "am", cmd_am, RUN_SETUP | NEED_WORK_TREE },
	{ "annotate", cmd_annotate, RUN_SETUP },
	{ "apply", cmd_apply, RUN_SETUP_GENTLY },
	{ "archive", cmd_archive, RUN_SETUP_GENTLY },
	{ "bisect--helper", cmd_bisect__helper, RUN_SETUP },
	{ "blame", cmd_blame, RUN_SETUP },
	{ "branch", cmd_branch, RUN_SETUP | DELAY_PAGER_CONFIG },
	{ "bundle", cmd_bundle, RUN_SETUP_GENTLY },
	{ "cat-file", cmd_cat_file, RUN_SETUP },
	{ "check-attr", cmd_check_attr, RUN_SETUP },
	{ "check-ignore", cmd_check_ignore, RUN_SETUP | NEED_WORK_TREE },
	{ "check-mailmap", cmd_check_mailmap, RUN_SETUP },
	{ "check-ref-format", cmd_check_ref_format },
	{ "checkout", cmd_checkout, RUN_SETUP | NEED_WORK_TREE },
	{ "checkout-index", cmd_checkout_index,
		RUN_SETUP | NEED_WORK_TREE},
	{ "cherry", cmd_cherry, RUN_SETUP },
	{ "cherry-pick", cmd_cherry_pick, RUN_SETUP | NEED_WORK_TREE },
	{ "clean", cmd_clean, RUN_SETUP | NEED_WORK_TREE },
	{ "clone", cmd_clone },
	{ "column", cmd_column, RUN_SETUP_GENTLY },
	{ "commit", cmd_commit, RUN_SETUP | NEED_WORK_TREE },
	{ "commit-tree", cmd_commit_tree, RUN_SETUP },
	{ "config", cmd_config, RUN_SETUP_GENTLY | DELAY_PAGER_CONFIG },
	{ "count-objects", cmd_count_objects, RUN_SETUP },
	{ "credential", cmd_credential, RUN_SETUP_GENTLY },
	{ "describe", cmd_describe, RUN_SETUP },
	{ "diff", cmd_diff },
	{ "diff-files", cmd_diff_files, RUN_SETUP | NEED_WORK_TREE },
	{ "diff-index", cmd_diff_index, RUN_SETUP },
	{ "diff-tree", cmd_diff_tree, RUN_SETUP },
	{ "difftool", cmd_difftool, RUN_SETUP | NEED_WORK_TREE },
	{ "fast-export", cmd_fast_export, RUN_SETUP },
	{ "fetch", cmd_fetch, RUN_SETUP },
	{ "fetch-pack", cmd_fetch_pack, RUN_SETUP },
	{ "fmt-merge-msg", cmd_fmt_merge_msg, RUN_SETUP },
	{ "for-each-ref", cmd_for_each_ref, RUN_SETUP },
	{ "format-patch", cmd_format_patch, RUN_SETUP },
	{ "fsck", cmd_fsck, RUN_SETUP },
	{ "fsck-objects", cmd_fsck, RUN_SETUP },
	{ "gc", cmd_gc, RUN_SETUP },
	{ "get-tar-commit-id", cmd_get_tar_commit_id },
	{ "grep", cmd_grep, RUN_SETUP_GENTLY },
	{ "hash-object", cmd_hash_object },
	{ "help", cmd_help },
	{ "index-pack", cmd_index_pack, RUN_SETUP_GENTLY },
	{ "init", cmd_init_db },
	{ "init-db", cmd_init_db },
	{ "interpret-trailers", cmd_interpret_trailers, RUN_SETUP_GENTLY },
	{ "log", cmd_log, RUN_SETUP },
	{ "ls-files", cmd_ls_files, RUN_SETUP },
	{ "ls-remote", cmd_ls_remote, RUN_SETUP_GENTLY },
	{ "ls-tree", cmd_ls_tree, RUN_SETUP },
	{ "mailinfo", cmd_mailinfo, RUN_SETUP_GENTLY },
	{ "mailsplit", cmd_mailsplit },
	{ "merge", cmd_merge, RUN_SETUP | NEED_WORK_TREE },
	{ "merge-base", cmd_merge_base, RUN_SETUP },
	{ "merge-file", cmd_merge_file, RUN_SETUP_GENTLY },
	{ "merge-index", cmd_merge_index, RUN_SETUP },
	{ "merge-ours", cmd_merge_ours, RUN_SETUP },
	{ "merge-recursive", cmd_merge_recursive, RUN_SETUP | NEED_WORK_TREE },
	{ "merge-recursive-ours", cmd_merge_recursive, RUN_SETUP | NEED_WORK_TREE },
	{ "merge-recursive-theirs", cmd_merge_recursive, RUN_SETUP | NEED_WORK_TREE },
	{ "merge-subtree", cmd_merge_recursive, RUN_SETUP | NEED_WORK_TREE },
	{ "merge-tree", cmd_merge_tree, RUN_SETUP },
	{ "mktag", cmd_mktag, RUN_SETUP },
	{ "mktree", cmd_mktree, RUN_SETUP },
	{ "mv", cmd_mv, RUN_SETUP | NEED_WORK_TREE },
	{ "name-rev", cmd_name_rev, RUN_SETUP },
	{ "notes", cmd_notes, RUN_SETUP },
	{ "pack-objects", cmd_pack_objects, RUN_SETUP },
	{ "pack-redundant", cmd_pack_redundant, RUN_SETUP },
	{ "pack-refs", cmd_pack_refs, RUN_SETUP },
	{ "patch-id", cmd_patch_id, RUN_SETUP_GENTLY },
	{ "pickaxe", cmd_blame, RUN_SETUP },
	{ "prune", cmd_prune, RUN_SETUP },
	{ "prune-packed", cmd_prune_packed, RUN_SETUP },
	{ "pull", cmd_pull, RUN_SETUP | NEED_WORK_TREE },
	{ "push", cmd_push, RUN_SETUP },
	{ "read-tree", cmd_read_tree, RUN_SETUP | SUPPORT_SUPER_PREFIX},
	{ "rebase--helper", cmd_rebase__helper, RUN_SETUP | NEED_WORK_TREE },
	{ "receive-pack", cmd_receive_pack },
	{ "reflog", cmd_reflog, RUN_SETUP },
	{ "remote", cmd_remote, RUN_SETUP },
	{ "remote-ext", cmd_remote_ext },
	{ "remote-fd", cmd_remote_fd },
	{ "repack", cmd_repack, RUN_SETUP },
	{ "replace", cmd_replace, RUN_SETUP },
	{ "rerere", cmd_rerere, RUN_SETUP },
	{ "reset", cmd_reset, RUN_SETUP },
	{ "rev-list", cmd_rev_list, RUN_SETUP },
	{ "rev-parse", cmd_rev_parse },
	{ "revert", cmd_revert, RUN_SETUP | NEED_WORK_TREE },
	{ "rm", cmd_rm, RUN_SETUP },
	{ "send-pack", cmd_send_pack, RUN_SETUP },
	{ "shortlog", cmd_shortlog, RUN_SETUP_GENTLY | USE_PAGER },
	{ "show", cmd_show, RUN_SETUP },
	{ "show-branch", cmd_show_branch, RUN_SETUP },
	{ "show-ref", cmd_show_ref, RUN_SETUP },
	{ "stage", cmd_add, RUN_SETUP | NEED_WORK_TREE },
	{ "status", cmd_status, RUN_SETUP | NEED_WORK_TREE },
	{ "stripspace", cmd_stripspace },
	{ "submodule--helper", cmd_submodule__helper, RUN_SETUP | SUPPORT_SUPER_PREFIX},
	{ "symbolic-ref", cmd_symbolic_ref, RUN_SETUP },
	{ "tag", cmd_tag, RUN_SETUP | DELAY_PAGER_CONFIG },
	{ "unpack-file", cmd_unpack_file, RUN_SETUP },
	{ "unpack-objects", cmd_unpack_objects, RUN_SETUP },
	{ "update-index", cmd_update_index, RUN_SETUP },
	{ "update-ref", cmd_update_ref, RUN_SETUP },
	{ "update-server-info", cmd_update_server_info, RUN_SETUP },
	{ "upload-archive", cmd_upload_archive },
	{ "upload-archive--writer", cmd_upload_archive_writer },
	{ "var", cmd_var, RUN_SETUP_GENTLY },
	{ "verify-commit", cmd_verify_commit, RUN_SETUP },
	{ "verify-pack", cmd_verify_pack },
	{ "verify-tag", cmd_verify_tag, RUN_SETUP },
	{ "version", cmd_version },
	{ "whatchanged", cmd_whatchanged, RUN_SETUP },
	{ "worktree", cmd_worktree, RUN_SETUP },
	{ "write-tree", cmd_write_tree, RUN_SETUP },
};
```

- run_builtin

命令执行代码为：

```C
status = p->fn(argc, argv, prefix);
```

只有 prefix 参数，目前不知道什么含义。

## References

- [Git Command Docs](https://git-scm.com/docs/)
