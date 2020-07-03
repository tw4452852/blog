---
categories:
- OSS
date: 2016-09-21T15:00:00
tags:
- linux
- concurrency
title: A memory corruption bug in ConsoleKit2
---

## background

这几天经常无法ssh登录到自己的服务器 , 通过htop发现console-kit-daemon这个
进程占用了100%的cpu , 问题应该就是它 .

## what is console-kit-daemon

首先 , 弄清楚它是什么 :

>ConsoleKit is a framework for keeping track of the various users, sessions, and seats
present on a system.  It provides a mechanism for software to react to changes of any
of these items or of any of the metadata associated with them.

也就是说 , 当通过ssh登录系统 , 从consolekit会为我们准备一个运行环境 , 
并通过一些抽象 ( session , seat ) 进行管理 . 当我们退出时 , 这些环境
将被销毁 . 

## ck-remove-dirtory

查看syslog , 发现每当退出时， 都会出现以下log :

```
traps: ck-remove-direc[23119] trap int3 ip:7f5d94016750 sp:7ffc5a95fd90 error:0
```

从名字来看 , 它好像用于删除一个目录 , 那么到底是删除哪个目录呢 ? 
又是谁调用它的呢 ? 
要回答这些问题 , 只有查看consolekit的代码 , 通过搜索 , 发现是在这里 :

```
static gboolean
remove_rundir (guint uid, const gchar *dest)
{
        gchar   *command;
        GError  *error = NULL;
        gboolean res;

        TRACE ();

        g_return_val_if_fail (dest, FALSE);

        if (uid < 1) {
                g_debug ("We didn't create a runtime dir for root, nothing to remove");
                return FALSE;
        }

        command = g_strdup_printf (LIBEXECDIR "/ck-remove-directory --uid=%d --dest=%s", uid, dest);

        res = g_spawn_command_line_sync (command, NULL, NULL, NULL, &error);

        if (! res) {
                g_warning ("Unable to remove user runtime dir '%s' error was: %s", dest, error->message);
                g_clear_error (&error);
                return FALSE;
        }

        return TRUE;
}
```

而调用它的地方有两处 , 首先是当用户退出时 :

```
gboolean
ck_remove_runtime_dir_for_user (guint uid)
{
        gchar        *dest;

        TRACE ();

        dest = get_rundir (uid);

        /* attempt to remove the tmpfs */
        ck_remove_tmpfs (uid, dest);

        /* remove the user's runtime dir now that all user sessions
         * are gone */
        remove_rundir (uid, dest);

        g_free (dest);

        return TRUE;
}
```

第二个新用户登录时为其准备环境时 , 发现之前旧的环境还没有被删除时 , 将其删除 :

```
gchar *
ck_generate_runtime_dir_for_user (guint uid)
{
	...

    dest = get_rundir (uid);

    /* Ensure any files from the last session are removed */
    if (g_file_test (dest, G_FILE_TEST_EXISTS) == TRUE) {
            remove_rundir (uid, dest);
    }
```

而且 , 删除的目录是用户的动态运行的临时目录 , 在目录在linux上就是
`/var/run/user/<uid>` , 而我用来登录的uid为1000 , 所以动态运行的临时目录
就是 `/var/run/user/1000` . 那我们手动调用 `ck-remove-directory` 来删除该目录 :

```
/usr/lib/ConsoleKit/ck-remove-directory --uid=1000 --dest=/var/run/user/1000

** (ck-remove-directory:28130): ERROR **: Failed to remove /var/run/user/1000, reason was: Permission denied
[1]    28130 trace trap  ./tools/ck-remove-directory --uid=1000 --dest=/var/run/user/1000
```

原来是permission deny , 那我们来看看该目录的权限 :

```
ls -ld /var/run/user/1000
drwx------ 2 tw root 40 Sep 21 15:06 /var/run/user/1000
```

它的父目录的权限 :

```
ls -ld /var/run/user/
drwxr-xr-x 3 root root 60 Sep 21 15:06 /var/run/user/
```

看来 , 的确时权限不够， `ck-remove-directory` 程序会以uid去删除该目录 . 
通过查看consolekit的issue , 发现已经有人碰到了同样的问题 , 同时也提供了
解决方法，
具体可以参见这个[PR](https://github.com/ConsoleKit2/ConsoleKit2/pull/67).
它的解决思路是以uid去删除 `/var/run/user/<uid>` 目录下的内容 , 然后再以调用
该程序的用户 ( 通常是root ) 去删除 `/var/run/user/<uid>` 目录本身.

## session hash table

当consolekit占用100%cpu时 , 通过gdb看看它在做什么:

```
#0 0x0000000000410bea in ck_seat_remove_session (seat=seat@entry=0x7fffe8011590, session=0x6680c0, error=error@entry=0x0) at ck-seat.c:701
#1 0x000000000040ab67 in remove_session_for_cookie (manager=0x6590f0, cookie=cookie@entry=0x65dae0 "totorow-1474285598.732068-49893253", error=error@entry=0x0) at ck-manager.c:3280
#2 0x000000000040b4e5 in remove_leader_for_connection (cookie=0x65dae0 "totorow-1474285598.732068-49893253", leader=0x64d4f0, data=0x7fffffffdf10) at ck-manager.c:3436
#3 0x00007ffff72f18d1 in g_hash_table_foreach_remove_or_steal (hash_table=0x652860, func=func@entry=0x40b4b0 <remove_leader_for_connection>, user_data=user_data@entry=0x7fffffffdf10, 
    notify=notify@entry=1) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/ghash.c:1491
#4 0x00007ffff72f298c in g_hash_table_foreach_remove (hash_table=<optimized out>, func=func@entry=0x40b4b0 <remove_leader_for_connection>, user_data=user_data@entry=0x7fffffffdf10)
    at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/ghash.c:1537
#5 0x000000000040c380 in remove_sessions_for_connection (service_name=<optimized out>, manager=0x6590f0) at ck-manager.c:3455
#6 on_name_owner_notify (connection=<optimized out>, sender_name=<optimized out>, object_path=<optimized out>, interface_name=<optimized out>, signal_name=<optimized out>, parameters=0x672aa0, 
    user_data=0x6590f0) at ck-manager.c:3491
#7 0x00007ffff790b9f5 in emit_signal_instance_in_idle_cb (data=0x7fffec0076f0) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/gio/gdbusconnection.c:3701
#8 0x00007ffff7302a55 in g_main_dispatch (context=0x64ac00) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3154
#9 g_main_context_dispatch (context=context@entry=0x64ac00) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3769
#10 0x00007ffff7302dc8 in g_main_context_iterate (context=0x64ac00, block=block@entry=1, dispatch=dispatch@entry=1, self=<optimized out>)
    at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3840
#11 0x00007ffff730308a in g_main_loop_run (loop=0x652d20) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:4034
#12 0x0000000000408840 in main (argc=1, argv=0x7fffffffe288) at main.c:307
```

通过查看代码 , 这里是从session hash table中删除对应的表项 . 

也就是说hash表被破坏了， 而且出问题之前， 也有如下log :

```
console-kit-daemon[29110]: GLib-CRITICAL: g_hash_table_foreach_remove_or_steal: assertion 'version == hash_table->version' failed
```

查看相关代码 ：

```
static guint
g_hash_table_foreach_remove_or_steal (GHashTable *hash_table,
                                      GHRFunc     func,
                                      gpointer    user_data,
                                      gboolean    notify)
{
  guint deleted = 0;
  gint i;
#ifndef G_DISABLE_ASSERT
  gint version = hash_table->version;
#endif

  for (i = 0; i < hash_table->size; i++)
    {
      guint node_hash = hash_table->hashes[i];
      gpointer node_key = hash_table->keys[i];
      gpointer node_value = hash_table->values[i];

      if (HASH_IS_REAL (node_hash) &&
          (* func) (node_key, node_value, user_data))
        {
          g_hash_table_remove_node (hash_table, i, notify);
          deleted++;
        }

#ifndef G_DISABLE_ASSERT
      g_return_val_if_fail (version == hash_table->version, 0);
#endif
    }

  g_hash_table_maybe_resize (hash_table);

#ifndef G_DISABLE_ASSERT
  if (deleted > 0)
    hash_table->version++;
#endif

  return deleted;
}
```

说明当删除hash表项时 , hash表发生了变化 . 
可能出现该问题 , 有可能是多线程的问题 , 也就是说多个线程同时在操作同一个
hash table . 通过查看代码 , 所有对该table的操作都在一个线程中 , 所以排除
该可能性 . 

通过在 `g_hash_table_foreach_remove_or_steal` 设置断点 , 出问题之前会有如下的
call stack :

```
#0  0x00007ffff72f18d1 in g_hash_table_foreach_remove_or_steal (hash_table=0x652860, func=func@entry=0x40b4b0 <remove_leader_for_connection>, user_data=user_data@entry=0x7fffffffd6c0, 
    notify=notify@entry=1) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/ghash.c:1491
#1  0x00007ffff72f298c in g_hash_table_foreach_remove (hash_table=<optimized out>, func=func@entry=0x40b4b0 <remove_leader_for_connection>, user_data=user_data@entry=0x7fffffffd6c0)
    at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/ghash.c:1537
#2  0x000000000040c380 in remove_sessions_for_connection (service_name=<optimized out>, manager=0x6590f0) at ck-manager.c:3455
#3  on_name_owner_notify (connection=<optimized out>, sender_name=<optimized out>, object_path=<optimized out>, interface_name=<optimized out>, signal_name=<optimized out>, 
    parameters=0x7fffec00d630, user_data=0x6590f0) at ck-manager.c:3491
#4  0x00007ffff790b9f5 in emit_signal_instance_in_idle_cb (data=0x7fffec00d580) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/gio/gdbusconnection.c:3701
#5  0x00007ffff7302a55 in g_main_dispatch (context=0x64ac00) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3154
#6  g_main_context_dispatch (context=context@entry=0x64ac00) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3769
#7  0x00007ffff7302dc8 in g_main_context_iterate (context=context@entry=0x64ac00, block=block@entry=1, dispatch=dispatch@entry=1, self=<optimized out>)
    at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3840
#8  0x00007ffff7302e6c in g_main_context_iteration (context=0x64ac00, context@entry=0x0, may_block=may_block@entry=1) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3901
#9 0x0000000000414c5c in ck_run_programs (dirpath=dirpath@entry=0x429f18 "/usr/lib64/ConsoleKit/run-session.d", action=action@entry=0x427037 "session_removed", 
    extra_env=extra_env@entry=0x7fffffffd8b0) at ck-run-programs.c:220
#10 0x000000000041416c in ck_session_run_programs (session=session@entry=0x6680c0, action=action@entry=0x427037 "session_removed") at ck-session.c:1330
#11 0x000000000040a553 in on_seat_session_removed_full (seat=0x7fffe8011590, session=0x6680c0, manager=0x6590f0) at ck-manager.c:2463
#12 0x00007ffff76030d7 in g_cclosure_marshal_VOID__OBJECTv (closure=0x670470, return_value=<optimized out>, instance=<optimized out>, args=<optimized out>, marshal_data=0x0, 
    n_params=<optimized out>, param_types=0x66c8f0) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/gobject/gmarshal.c:2102
#13 0x00007ffff7600237 in _g_closure_invoke_va (closure=closure@entry=0x670470, return_value=return_value@entry=0x0, instance=instance@entry=0x7fffe8011590, args=args@entry=0x7fffffffdc78, 
    n_params=1, param_types=0x66c8f0) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/gobject/gclosure.c:864
#14 0x00007ffff7618f88 in g_signal_emit_valist (instance=0x7fffe8011590, signal_id=<optimized out>, detail=0, var_args=var_args@entry=0x7fffffffdc78)
    at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/gobject/gsignal.c:3292
#15 0x00007ffff7619c0a in g_signal_emit (instance=instance@entry=0x7fffe8011590, signal_id=<optimized out>, detail=detail@entry=0)
    at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/gobject/gsignal.c:3439
#16 0x0000000000410bea in ck_seat_remove_session (seat=seat@entry=0x7fffe8011590, session=0x6680c0, error=error@entry=0x0) at ck-seat.c:701
#17 0x000000000040ab67 in remove_session_for_cookie (manager=0x6590f0, cookie=cookie@entry=0x65dae0 "totorow-1474285598.732068-49893253", error=error@entry=0x0) at ck-manager.c:3280
#18 0x000000000040b4e5 in remove_leader_for_connection (cookie=0x65dae0 "totorow-1474285598.732068-49893253", leader=0x64d4f0, data=0x7fffffffdf10) at ck-manager.c:3436
#19 0x00007ffff72f18d1 in g_hash_table_foreach_remove_or_steal (hash_table=0x652860, func=func@entry=0x40b4b0 <remove_leader_for_connection>, user_data=user_data@entry=0x7fffffffdf10, 
    notify=notify@entry=1) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/ghash.c:1491
#20 0x00007ffff72f298c in g_hash_table_foreach_remove (hash_table=<optimized out>, func=func@entry=0x40b4b0 <remove_leader_for_connection>, user_data=user_data@entry=0x7fffffffdf10)
    at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/ghash.c:1537
#21 0x000000000040c380 in remove_sessions_for_connection (service_name=<optimized out>, manager=0x6590f0) at ck-manager.c:3455
#22 on_name_owner_notify (connection=<optimized out>, sender_name=<optimized out>, object_path=<optimized out>, interface_name=<optimized out>, signal_name=<optimized out>, parameters=0x672aa0, 
    user_data=0x6590f0) at ck-manager.c:3491
#23 0x00007ffff790b9f5 in emit_signal_instance_in_idle_cb (data=0x7fffec0076f0) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/gio/gdbusconnection.c:3701
#24 0x00007ffff7302a55 in g_main_dispatch (context=0x64ac00) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3154
#25 g_main_context_dispatch (context=context@entry=0x64ac00) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3769
#26 0x00007ffff7302dc8 in g_main_context_iterate (context=0x64ac00, block=block@entry=1, dispatch=dispatch@entry=1, self=<optimized out>)
    at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:3840
#27 0x00007ffff730308a in g_main_loop_run (loop=0x652d20) at /var/tmp/portage/dev-libs/glib-2.46.2-r3/work/glib-2.46.2/glib/gmain.c:4034
#28 0x0000000000408840 in main (argc=1, argv=0x7fffffffe288) at main.c:307
```

原来 , 这里出现了recursive call !!!
通过分析该部分代码 , 当从一个hash
table删除一个session时，会执行一些其他废时操作 , 
为了性能的考虑 , 在这些操作执行时 , 主线程仍然可以处理其他请求 , 
所以当一个session还没有完全删除时 , 另一个session被删除了 ,
这样， hash table有可能被resize , 那么就会触发之前的assertion . 
那么， 原来的操作仍会使用已经被释放的hash table . 所以一些unexpected结果就会发生. 

## conclusion

为了防止recursive call， 解决的思路是先将从hash table中将相关的表项删除,
将相关的清理操作放在之后执行， 这样即使出现了recursive call发生 , 
对hash table的操作也不会被破坏 , 具体参见该
[PR](https://github.com/ConsoleKit2/ConsoleKit2/pull/81). 问题解决 ! 

FIN.
