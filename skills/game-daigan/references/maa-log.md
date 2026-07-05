# MAA Log-Based Completion Detection

MAA writes task progress and completion signals to `debug/gui.log`.

## Log File Path

```
{install_dir}/debug/gui.log
```

For this machine (Scoop install):

```
~/scoop/apps/maa/current/debug/gui.log
```

## Completion Keywords

When all queued tasks are done, the log ends with:

```
任务已全部完成！
(用时 0h XXm XXs)
理智将在 YYYY-MM-DD HH:MM 回满。(XXh XXm 后)
```

The preceding lines show each completed task chain:

```
<2> 完成任务: 基建换班
<2> 完成任务: 领取奖励
<2> 完成任务: 更新数据
...
<2> 任务已全部完成！
```

## How to Check

```bash
# Read the last ~30 lines
tail -n 30 ~/scoop/apps/maa/current/debug/gui.log

# Quick check for completion signal
grep "任务已全部完成" ~/scoop/apps/maa/current/debug/gui.log
```

## When Log Doesn't Show Completion Yet

- MAA is still running tasks — poll every 30–60 seconds
- If `gui.log` contains recent task entries but no "任务已全部完成", tasks are still in progress
- Fall back to vision-based monitoring if the log file is inaccessible

## Pitfalls

- **Log buffer delay**: MAA may flush logs with a slight delay. Wait a few seconds between polls.
- **File locking**: `gui.log` is written by `MAA.exe` while running. `tail` and `grep` can still read it safely on Windows.
- **Multiple days' runs**: The log accumulates across sessions. Always check the **last** occurrence of the completion keyword, or tail the end of the file.
- **Task queue changes**: If you change the task list mid-run (e.g. add new tasks), the completion signal reflects the current queue only.