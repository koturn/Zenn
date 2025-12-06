---
title: "同時実行数が指定可能なTaskScheduler"
emoji: "⚖️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp"]
published: true
---

## 同時実行数が指定可能なTaskScheduler

[MSDNには同時実行数が指定可能なTaskSchedulerの実装例](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler "TaskScheduler Class (System.Threading.Tasks) | Microsoft Learn")が示されている．
この実装はそのままでも有用ではあるが，少々コードが古い C# と .NET 向けのものになっているので，

- nullable対応
- ロックには .NET 9.0 のLockクラスを使用
- XMLドキュメンテーションコメント付与

という対応を行うと下記のようになる．

```cs:LimitedConcurrencyLevelTaskScheduler.cs
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;


namespace Koturn.Threading.Tasks
{
    /// <summary>
    /// Provides a task scheduler that ensures a maximum concurrency level while running on top of the thread pool.
    /// </summary>
    /// <remarks>
    /// <seealso href="https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler"/>
    /// </remarks>
    public sealed class LimitedConcurrencyLevelTaskScheduler : TaskScheduler
    {
        /// <summary>
        /// Indicates whether the current thread is processing work items.
        /// </summary>
        [ThreadStatic]
        private static bool _currentThreadIsProcessingItems;

        /// <summary>
        /// The maximum concurrency level allowed by this scheduler.
        /// </summary>
        public sealed override int MaximumConcurrencyLevel => _maxDegreeOfParallelism;

        /// <summary>
        /// The list of tasks to be executed.
        /// </summary>
        private readonly LinkedList<Task> _tasks = new();
        /// <summary>
        /// The maximum concurrency level allowed by this scheduler.
        /// </summary>
        private int _maxDegreeOfParallelism;
        /// <summary>
        /// <para>Indicates whether the scheduler is currently processing work items.</para>
        /// <para>This variable locked by <see cref="_taskListLock"/>.</para>
        /// </summary>
        private int _delegatesQueuedOrRunning = 0;
        /// <summary>
        /// Lock object for <see cref="_tasks"/> and <see cref="_delegatesQueuedOrRunning"/>.
        /// </summary>
#if NET9_0_OR_GREATER
        private readonly Lock _taskListLock = new();
#else
        private readonly object _taskListLock = new();
#endif  // NET9_0_OR_GREATER


        /// <summary>
        /// Creates a new instance with the specified degree of parallelism.
        /// </summary>
        /// <param name="maxDegreeOfParallelism"></param>
        public LimitedConcurrencyLevelTaskScheduler(int maxDegreeOfParallelism)
        {
#if NET8_0_OR_GREATER
            ArgumentOutOfRangeException.ThrowIfLessThan(maxDegreeOfParallelism, 1);
#else
            ThrowIfLessThan(maxDegreeOfParallelism, 1, nameof(maxDegreeOfParallelism));
#endif  // NET8_0_OR_GREATER
            _maxDegreeOfParallelism = maxDegreeOfParallelism;
        }


        /// <summary>
        /// Queues a task to the scheduler.
        /// </summary>
        /// <param name="task">A task.</param>
        protected sealed override void QueueTask(Task task)
        {
            // Add the task to the list of tasks to be processed. If there aren't enough
            // delegates currently queued or running to process tasks, schedule another.
            lock (_taskListLock)
            {
                _tasks.AddLast(task);
                if (_delegatesQueuedOrRunning < _maxDegreeOfParallelism)
                {
                    _delegatesQueuedOrRunning++;
                    NotifyThreadPoolOfPendingWork();
                }
            }
        }

        /// <summary>
        /// Attempts to execute the specified task on the current thread.
        /// </summary>
        /// <param name="task">A task to execute.</param>
        /// <param name="taskWasPreviouslyQueued">A flag whether <paramref name="task"/> is queued previously.</param>
        /// <returns><c>true</c> if task was successfully executed, <c>false</c> if it was not.</returns>
        protected sealed override bool TryExecuteTaskInline(Task task, bool taskWasPreviouslyQueued)
        {
            // If this thread isn't already processing a task, we don't support inlining.
            // If the task was previously queued, remove it from the queue.
            return _currentThreadIsProcessingItems
                && (!taskWasPreviouslyQueued || TryDequeue(task))
                && TryExecuteTask(task);
        }

        /// <summary>
        /// Attempt to remove a previously scheduled task from the scheduler.
        /// </summary>
        /// <param name="task">A task to remove from the scheduler</param>
        /// <returns><c>true</c> if the element containing value is successfully removed; otherwise, <c>false</c>.</returns>
        protected sealed override bool TryDequeue(Task task)
        {
            lock (_taskListLock)
            {
                return _tasks.Remove(task);
            }
        }

        /// <summary>
        /// Gets an enumerable of the tasks currently scheduled on this scheduler.
        /// </summary>
        /// <returns>An enumerator of tasks.</returns>
        protected sealed override IEnumerable<Task> GetScheduledTasks()
        {
            var lockTaken = false;

#if NET9_0_OR_GREATER
            try
            {
                lockTaken = _taskListLock.TryEnter();
                if (!lockTaken)
                {
                    throw new NotSupportedException();
                }
                return _tasks;
            }
            finally
            {
                if (lockTaken)
                {
                    _taskListLock.Exit();
                }
            }
#else
            try
            {
                Monitor.TryEnter(_taskListLock, ref lockTaken);
                if (!lockTaken)
                {
                    throw new NotSupportedException();
                }
                return _tasks;
            }
            finally
            {
                if (lockTaken)
                {
                    Monitor.Exit(_taskListLock);
                }
            }
#endif  // NET9_0_OR_GREATER
        }


        /// <summary>
        /// Inform the <see cref="ThreadPool"/> that there's work to be executed for this scheduler.
        /// </summary>
        private void NotifyThreadPoolOfPendingWork()
        {
            ThreadPool.UnsafeQueueUserWorkItem(_ =>
            {
                // Note that the current thread is now processing work items.
                // This is necessary to enable inlining of tasks into this thread.
                _currentThreadIsProcessingItems = true;
                try
                {
                    // Process all available items in the queue.
                    while (true)
                    {
                        Task task;
                        lock (_taskListLock)
                        {
                            // When there are no more items to be processed,
                            // note that we're done processing, and get out.
                            var node = _tasks.First;
                            if (node == null)
                            {
                                _delegatesQueuedOrRunning--;
                                break;
                            }
                            // Get the next item from the queue.
                            task = node.Value;
                            _tasks.RemoveFirst();
                        }
                        // Execute the task we pulled out of the queue.
                        TryExecuteTask(task);
                    }
                }
                // We're done processing items on the current thread.
                finally
                {
                    _currentThreadIsProcessingItems = false;
                }
            }, null);
        }


#if !NET8_0_OR_GREATER
        /// <summary>
        /// Throw <see cref="ArgumentOutOfRangeException"/>.
        /// </summary>
        /// <typeparam name="T">The type of the objects.</typeparam>
        /// <param name="value">The maxDegreeOfParallelism of the argument that causes this exception.</param>
        /// <param name="other">The maxDegreeOfParallelism to compare with <paramref name="value"/>.</param>
        /// <param name="paramName">The name of the parameter with which <paramref name="value"/> corresponds.</param>
        /// <exception cref="ArgumentOutOfRangeException">Always thrown.</exception>
#if NETCOREAPP3_0_OR_GREATER || NETSTANDARD2_1
        [DoesNotReturn]
#endif  // NETCOREAPP3_0_OR_GREATER || NETSTANDARD2_1
        private static void ThrowLess<T>(T value, T other, string paramName)
        {
            throw new ArgumentOutOfRangeException(paramName, value, $"'{value}' must be greater than or equal to '{other}'.");
        }

        /// <summary>
        /// Throws an <see cref="ArgumentOutOfRangeException"/> if <paramref name="value"/> is less than <paramref name="other"/>.
        /// </summary>
        /// <typeparam name="T">The type of the objects to validate.</typeparam>
        /// <param name="value">The argument to validate as greater than or equal to <paramref name="other"/>.</param>
        /// <param name="other">The maxDegreeOfParallelism to compare with <paramref name="value"/>.</param>
        /// <param name="paramName">The name of the parameter with which <paramref name="value"/> corresponds.</param>
        /// <exception cref="ArgumentOutOfRangeException">Thrown if <paramref name="value"/> is less than <paramref name="other"/>.</exception>
        internal static void ThrowIfLessThan<T>(T value, T other, string paramName)
            where T : IComparable<T>
        {
            if (value.CompareTo(other) < 0)
            {
                ThrowLess(value, other, paramName);
            }
        }
#endif  // !NET8_0_OR_GREATER
    }
}
```

## 後から同時実行数を変更できるTaskScheduler

前述のTaskSchedulerはインスタンス生成時に同時実行数を指定するもので，後から変更することができなかった．
後から変更することはそう多くはないかもしれないが，並列に実行しても長時間要する処理である場合にはニーズがあると考えられる．
例えば，1ファイルあたり1分かかる処理を1000ファイルを対象に8並列で実施するとすれば125分要する．
125分を待ち切れず途中でゲームをしたくなった場合，並列数を減らしたくなるかもしれない．

後から同時実行数を変更可能なTaskSchedulerは下記のとおりとなる．

```cs:LimitedConcurrencyLevelTaskScheduler.cs
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;


namespace Koturn.Threading.Tasks
{
    /// <summary>
    /// Provides a task scheduler that ensures a maximum concurrency level while running on top of the thread pool.
    /// </summary>
    /// <remarks>
    /// <seealso href="https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler"/>
    /// </remarks>
    public sealed class LimitedConcurrencyLevelTaskScheduler : TaskScheduler
    {
        /// <summary>
        /// Indicates whether the current thread is processing work items.
        /// </summary>
        [ThreadStatic]
        private static bool _currentThreadIsProcessingItems;

        /// <summary>
        /// The maximum concurrency level allowed by this scheduler.
        /// </summary>
        public sealed override int MaximumConcurrencyLevel => _maxDegreeOfParallelism;

        /// <summary>
        /// The list of tasks to be executed.
        /// </summary>
        private readonly LinkedList<Task> _tasks = new();
        /// <summary>
        /// The maximum concurrency level allowed by this scheduler.
        /// </summary>
        private int _maxDegreeOfParallelism;
        /// <summary>
        /// <para>Indicates whether the scheduler is currently processing work items.</para>
        /// <para>This variable locked by <see cref="_taskListLock"/>.</para>
        /// </summary>
        private int _delegatesQueuedOrRunning = 0;
        /// <summary>
        /// Lock object for <see cref="_tasks"/> and <see cref="_delegatesQueuedOrRunning"/>.
        /// </summary>
#if NET9_0_OR_GREATER
        private readonly Lock _taskListLock = new();
#else
        private readonly object _taskListLock = new();
#endif  // NET9_0_OR_GREATER


        /// <summary>
        /// Creates a new instance with the specified degree of parallelism.
        /// </summary>
        /// <param name="maxDegreeOfParallelism"></param>
        public LimitedConcurrencyLevelTaskScheduler(int maxDegreeOfParallelism)
        {
#if NET8_0_OR_GREATER
            ArgumentOutOfRangeException.ThrowIfLessThan(maxDegreeOfParallelism, 1);
#else
            ThrowIfLessThan(maxDegreeOfParallelism, 1, nameof(maxDegreeOfParallelism));
#endif  // NET8_0_OR_GREATER
            _maxDegreeOfParallelism = maxDegreeOfParallelism;
        }


        /// <summary>
        /// Changes <see cref="MaximumConcurrencyLevel"/>.
        /// </summary>
        /// <param name="maxDegreeOfParallelism"></param>
        public void SetMaximumConcurrencyLevel(int maxDegreeOfParallelism)
        {
#if NET8_0_OR_GREATER
            ArgumentOutOfRangeException.ThrowIfLessThan(maxDegreeOfParallelism, 1);
#else
            ThrowIfLessThan(maxDegreeOfParallelism, 1, nameof(maxDegreeOfParallelism));
#endif  // NET8_0_OR_GREATER
            var maxDegreeOfParallelismOld = _maxDegreeOfParallelism;
            _maxDegreeOfParallelism = maxDegreeOfParallelism;

            if (maxDegreeOfParallelismOld >= maxDegreeOfParallelism)
            {
                return;
            }

            lock (_taskListLock)
            {
                var diff = Math.Min(maxDegreeOfParallelism, _tasks.Count) - _delegatesQueuedOrRunning;
                if (diff > 0)
                {
                    _delegatesQueuedOrRunning += diff;
                    for (int i = 0; i < diff; i++)
                    {
                        NotifyThreadPoolOfPendingWork();
                    }
                }
            }
        }


        /// <summary>
        /// Queues a task to the scheduler.
        /// </summary>
        /// <param name="task">A task.</param>
        protected sealed override void QueueTask(Task task)
        {
            // Add the task to the list of tasks to be processed. If there aren't enough
            // delegates currently queued or running to process tasks, schedule another.
            lock (_taskListLock)
            {
                _tasks.AddLast(task);
                if (_delegatesQueuedOrRunning < _maxDegreeOfParallelism)
                {
                    _delegatesQueuedOrRunning++;
                    NotifyThreadPoolOfPendingWork();
                }
            }
        }

        /// <summary>
        /// Attempts to execute the specified task on the current thread.
        /// </summary>
        /// <param name="task">A task to execute.</param>
        /// <param name="taskWasPreviouslyQueued">A flag whether <paramref name="task"/> is queued previously.</param>
        /// <returns><c>true</c> if task was successfully executed, <c>false</c> if it was not.</returns>
        protected sealed override bool TryExecuteTaskInline(Task task, bool taskWasPreviouslyQueued)
        {
            // If this thread isn't already processing a task, we don't support inlining.
            // If the task was previously queued, remove it from the queue.
            return _currentThreadIsProcessingItems
                && (!taskWasPreviouslyQueued || TryDequeue(task))
                && TryExecuteTask(task);
        }

        /// <summary>
        /// Attempt to remove a previously scheduled task from the scheduler.
        /// </summary>
        /// <param name="task">A task to remove from the scheduler</param>
        /// <returns><c>true</c> if the element containing value is successfully removed; otherwise, <c>false</c>.</returns>
        protected sealed override bool TryDequeue(Task task)
        {
            lock (_taskListLock)
            {
                return _tasks.Remove(task);
            }
        }

        /// <summary>
        /// Gets an enumerable of the tasks currently scheduled on this scheduler.
        /// </summary>
        /// <returns>An enumerator of tasks.</returns>
        protected sealed override IEnumerable<Task> GetScheduledTasks()
        {
            var lockTaken = false;

#if NET9_0_OR_GREATER
            try
            {
                lockTaken = _taskListLock.TryEnter();
                if (!lockTaken)
                {
                    throw new NotSupportedException();
                }
                return _tasks;
            }
            finally
            {
                if (lockTaken)
                {
                    _taskListLock.Exit();
                }
            }
#else
            try
            {
                Monitor.TryEnter(_taskListLock, ref lockTaken);
                if (!lockTaken)
                {
                    throw new NotSupportedException();
                }
                return _tasks;
            }
            finally
            {
                if (lockTaken)
                {
                    Monitor.Exit(_taskListLock);
                }
            }
#endif  // NET9_0_OR_GREATER
        }


        /// <summary>
        /// Inform the <see cref="ThreadPool"/> that there's work to be executed for this scheduler.
        /// </summary>
        private void NotifyThreadPoolOfPendingWork()
        {
            ThreadPool.UnsafeQueueUserWorkItem(_ =>
            {
                // Note that the current thread is now processing work items.
                // This is necessary to enable inlining of tasks into this thread.
                _currentThreadIsProcessingItems = true;
                try
                {
                    // Process all available items in the queue.
                    while (true)
                    {
                        Task task;
                        lock (_taskListLock)
                        {
                            // Terminate tasks exceeding the maximum concurrency level.
                            if (_delegatesQueuedOrRunning > _maxDegreeOfParallelism)
                            {
                                _delegatesQueuedOrRunning--;
                                break;
                            }
                            // When there are no more items to be processed,
                            // note that we're done processing, and get out.
                            var node = _tasks.First;
                            if (node == null)
                            {
                                _delegatesQueuedOrRunning--;
                                break;
                            }
                            // Get the next item from the queue.
                            task = node.Value;
                            _tasks.RemoveFirst();
                        }
                        // Execute the task we pulled out of the queue.
                        TryExecuteTask(task);
                    }
                }
                // We're done processing items on the current thread.
                finally
                {
                    _currentThreadIsProcessingItems = false;
                }
            }, null);
        }


#if !NET8_0_OR_GREATER
        /// <summary>
        /// Throw <see cref="ArgumentOutOfRangeException"/>.
        /// </summary>
        /// <typeparam name="T">The type of the objects.</typeparam>
        /// <param name="value">The maxDegreeOfParallelism of the argument that causes this exception.</param>
        /// <param name="other">The maxDegreeOfParallelism to compare with <paramref name="value"/>.</param>
        /// <param name="paramName">The name of the parameter with which <paramref name="value"/> corresponds.</param>
        /// <exception cref="ArgumentOutOfRangeException">Always thrown.</exception>
#if NETCOREAPP3_0_OR_GREATER || NETSTANDARD2_1
        [DoesNotReturn]
#endif  // NETCOREAPP3_0_OR_GREATER || NETSTANDARD2_1
        private static void ThrowLess<T>(T value, T other, string paramName)
        {
            throw new ArgumentOutOfRangeException(paramName, value, $"'{value}' must be greater than or equal to '{other}'.");
        }

        /// <summary>
        /// Throws an <see cref="ArgumentOutOfRangeException"/> if <paramref name="value"/> is less than <paramref name="other"/>.
        /// </summary>
        /// <typeparam name="T">The type of the objects to validate.</typeparam>
        /// <param name="value">The argument to validate as greater than or equal to <paramref name="other"/>.</param>
        /// <param name="other">The maxDegreeOfParallelism to compare with <paramref name="value"/>.</param>
        /// <param name="paramName">The name of the parameter with which <paramref name="value"/> corresponds.</param>
        /// <exception cref="ArgumentOutOfRangeException">Thrown if <paramref name="value"/> is less than <paramref name="other"/>.</exception>
        internal static void ThrowIfLessThan<T>(T value, T other, string paramName)
            where T : IComparable<T>
        {
            if (value.CompareTo(other) < 0)
            {
                ThrowLess(value, other, paramName);
            }
        }
#endif  // !NET8_0_OR_GREATER
    }
}
```

変更差分は大きくなく，下記2箇所のみ．

```diff
diff --git LimitedConcurrencyLevelTaskScheduler.cs LimitedConcurrencyLevelTaskScheduler.cs
index 991d329..5c82d40 100644
--- LimitedConcurrencyLevelTaskScheduler.cs
+++ LimitedConcurrencyLevelTaskScheduler.cs
@@ -63,6 +63,40 @@ namespace Koturn.Threading.Tasks
         }


+        /// <summary>
+        /// Changes <see cref="MaximumConcurrencyLevel"/>.
+        /// </summary>
+        /// <param name="maxDegreeOfParallelism"></param>
+        public void SetMaximumConcurrencyLevel(int maxDegreeOfParallelism)
+        {
+#if NET8_0_OR_GREATER
+            ArgumentOutOfRangeException.ThrowIfLessThan(maxDegreeOfParallelism, 1);
+#else
+            ThrowIfLessThan(maxDegreeOfParallelism, 1, nameof(maxDegreeOfParallelism));
+#endif  // NET8_0_OR_GREATER
+            var maxDegreeOfParallelismOld = _maxDegreeOfParallelism;
+            _maxDegreeOfParallelism = maxDegreeOfParallelism;
+
+            if (maxDegreeOfParallelismOld >= maxDegreeOfParallelism)
+            {
+                return;
+            }
+
+            lock (_taskListLock)
+            {
+                var diff = Math.Min(maxDegreeOfParallelism, _tasks.Count) - _delegatesQueuedOrRunning;
+                if (diff > 0)
+                {
+                    _delegatesQueuedOrRunning += diff;
+                    for (int i = 0; i < diff; i++)
+                    {
+                        NotifyThreadPoolOfPendingWork();
+                    }
+                }
+            }
+        }
+
+
         /// <summary>
         /// Queues a task to the scheduler.
         /// </summary>
@@ -174,6 +208,12 @@ namespace Koturn.Threading.Tasks
                         Task task;
                         lock (_taskListLock)
                         {
+                            // Terminate tasks exceeding the maximum concurrency level.
+                            if (_delegatesQueuedOrRunning > _maxDegreeOfParallelism)
+                            {
+                                _delegatesQueuedOrRunning--;
+                                break;
+                            }
                             // When there are no more items to be processed,
                             // note that we're done processing, and get out.
                             var node = _tasks.First;
```

`MaximumConcurrencyLevel` は親クラスの設計に依存し，getのみのプロパティであるため，別途設定用のメソッドである `SetMaximumConcurrencyLevel()` を設けた．
同時実行数を増やした場合，ペンディングとなっているタスクを即座に実行するようにしてある．

反対に同時実行数を減らした場合は， `ThreadPool.UnsafeQueueUserWorkItem()` に与えるラムダ式にて，次の実行タスクを取り出す前に，現在のタスク数と設定されている同時実行数を比較し，超過していればそのスレッドを終了する処理を加えている．
現在実行中のTaskを停止させるのではなく，次のタスクの取り出しの際にスレッド数を減らす作りのため，同時実行数を減らす場合の反映はラグがある．


## TaskCreationOptions.LongRunning のサポート
実はここまでの `LimitedConcurrencyLevelTaskScheduler` の実装では `TaskCreationOptions.LongRunning` が指定されたタスクのサポートが不十分であり，スレッドプールのスレッドで実行されてしまう．
デフォルトのタスクスケジュータでは新規にスレッドを作成し，そのスレッド上で実行するようになっているので，それに習うと下記のような実装になる．

```cs:LimitedConcurrencyLevelTaskScheduler.cs
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;


namespace Koturn.Threading.Tasks
{
    /// <summary>
    /// Provides a task scheduler that ensures a maximum concurrency level while running on top of the thread pool.
    /// </summary>
    /// <remarks>
    /// <seealso href="https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler"/>
    /// </remarks>
    public sealed class LimitedConcurrencyLevelTaskScheduler : TaskScheduler
    {
        /// <summary>
        /// Indicates whether the current thread is processing work items.
        /// </summary>
        [ThreadStatic]
        private static bool _currentThreadIsProcessingItems;

        /// <summary>
        /// The maximum concurrency level allowed by this scheduler.
        /// </summary>
        public sealed override int MaximumConcurrencyLevel => _maxDegreeOfParallelism;

        /// <summary>
        /// The list of tasks to be executed.
        /// </summary>
        private readonly LinkedList<Task> _tasks = new();
        /// <summary>
        /// The maximum concurrency level allowed by this scheduler.
        /// </summary>
        private int _maxDegreeOfParallelism;
        /// <summary>
        /// <para>Indicates whether the scheduler is currently processing work items.</para>
        /// <para>This variable locked by <see cref="_taskListLock"/>.</para>
        /// </summary>
        private int _delegatesQueuedOrRunning = 0;
        /// <summary>
        /// Lock object for <see cref="_tasks"/> and <see cref="_delegatesQueuedOrRunning"/>.
        /// </summary>
#if NET9_0_OR_GREATER
        private readonly Lock _taskListLock = new();
#else
        private readonly object _taskListLock = new();
#endif  // !NET9_0_OR_GREATER
        /// <summary>
        /// True to allow non long running task in new (non pooled) thread.
        /// </summary>
        private readonly bool _allowTORunNonLongRunningTaskOnNewThread;


        /// <summary>
        /// Creates a new instance with the specified degree of parallelism.
        /// </summary>
        /// <param name="maxDegreeOfParallelism"></param>
        /// <param name="allowRunNonLongRunningTaskInNonPooledThread">True to allow non long running task in new (non pooled) thread.</param>
        public LimitedConcurrencyLevelTaskScheduler(int maxDegreeOfParallelism, bool allowRunNonLongRunningTaskInNonPooledThread = true)
        {
#if NET8_0_OR_GREATER
            ArgumentOutOfRangeException.ThrowIfLessThan(maxDegreeOfParallelism, 1);
#else
            ThrowIfLessThan(maxDegreeOfParallelism, 1, nameof(maxDegreeOfParallelism));
#endif  // NET8_0_OR_GREATER
            _maxDegreeOfParallelism = maxDegreeOfParallelism;
            _allowTORunNonLongRunningTaskOnNewThread = allowRunNonLongRunningTaskInNonPooledThread;
        }


        /// <summary>
        /// Changes <see cref="MaximumConcurrencyLevel"/>.
        /// </summary>
        /// <param name="maxDegreeOfParallelism"></param>
        public void SetMaximumConcurrencyLevel(int maxDegreeOfParallelism)
        {
#if NET8_0_OR_GREATER
            ArgumentOutOfRangeException.ThrowIfLessThan(maxDegreeOfParallelism, 1);
#else
            ThrowIfLessThan(maxDegreeOfParallelism, 1, nameof(maxDegreeOfParallelism));
#endif  // NET8_0_OR_GREATER
            var maxDegreeOfParallelismOld = _maxDegreeOfParallelism;
            _maxDegreeOfParallelism = maxDegreeOfParallelism;

            if (maxDegreeOfParallelismOld >= maxDegreeOfParallelism)
            {
                return;
            }

            lock (_taskListLock)
            {
                var diff = Math.Min(maxDegreeOfParallelism, _tasks.Count) - _delegatesQueuedOrRunning;
                if (diff > 0)
                {
                    _delegatesQueuedOrRunning += diff;
                    for (int i = 0; i < diff; i++)
                    {
                        NotifyThreadPoolOfPendingWork();
                    }
                }
            }
        }


        /// <summary>
        /// Queues a task to the scheduler.
        /// </summary>
        /// <param name="task">A task.</param>
        protected sealed override void QueueTask(Task task)
        {
            lock (_taskListLock)
            {
                if (_delegatesQueuedOrRunning < _maxDegreeOfParallelism)
                {
                    _delegatesQueuedOrRunning++;
                    if ((task.CreationOptions & TaskCreationOptions.LongRunning) == TaskCreationOptions.None)
                    {
                        // Add the task to the list of tasks to be processed.
                        _tasks.AddLast(task);
                        NotifyThreadPoolOfPendingWork();
                    }
                    else
                    {
                        // Run the long running task on the new thread.
                        RunTaskOnNewThread(task);
                    }
                }
                else
                {
                    // If there aren't enough delegates currently queued or running to process tasks, schedule another.
                    _tasks.AddLast(task);
                }
            }
        }

        /// <summary>
        /// Attempts to execute the specified task on the current thread.
        /// </summary>
        /// <param name="task">A task to execute.</param>
        /// <param name="taskWasPreviouslyQueued">A flag whether <paramref name="task"/> is queued previously.</param>
        /// <returns><c>true</c> if task was successfully executed, <c>false</c> if it was not.</returns>
        protected sealed override bool TryExecuteTaskInline(Task task, bool taskWasPreviouslyQueued)
        {
            // If this thread isn't already processing a task, we don't support inlining.
            // If the task was previously queued, remove it from the queue.
            return _currentThreadIsProcessingItems
                && (!taskWasPreviouslyQueued || TryDequeue(task))
                && TryExecuteTask(task);
        }

        /// <summary>
        /// Attempt to remove a previously scheduled task from the scheduler.
        /// </summary>
        /// <param name="task">A task to remove from the scheduler</param>
        /// <returns><c>true</c> if the element containing value is successfully removed; otherwise, <c>false</c>.</returns>
        protected sealed override bool TryDequeue(Task task)
        {
            lock (_taskListLock)
            {
                return _tasks.Remove(task);
            }
        }

        /// <summary>
        /// Gets an enumerable of the tasks currently scheduled on this scheduler.
        /// </summary>
        /// <returns>An enumerator of tasks.</returns>
        protected sealed override IEnumerable<Task> GetScheduledTasks()
        {
            var lockTaken = false;

#if NET9_0_OR_GREATER
            try
            {
                lockTaken = _taskListLock.TryEnter();
                if (!lockTaken)
                {
                    throw new NotSupportedException();
                }
                return _tasks;
            }
            finally
            {
                if (lockTaken)
                {
                    _taskListLock.Exit();
                }
            }
#else
            try
            {
                Monitor.TryEnter(_taskListLock, ref lockTaken);
                if (!lockTaken)
                {
                    throw new NotSupportedException();
                }
                return _tasks;
            }
            finally
            {
                if (lockTaken)
                {
                    Monitor.Exit(_taskListLock);
                }
            }
#endif  // NET9_0_OR_GREATER
        }


        /// <summary>
        /// Inform the <see cref="ThreadPool"/> that there's work to be executed for this scheduler.
        /// </summary>
        private void NotifyThreadPoolOfPendingWork()
        {
            ThreadPool.UnsafeQueueUserWorkItem(_ =>
            {
                // Note that the current thread is now processing work items.
                // This is necessary to enable inlining of tasks into this thread.
                _currentThreadIsProcessingItems = true;
                try
                {
                    // Process all available items in the queue.
                    while (true)
                    {
                        Task task;
                        lock (_taskListLock)
                        {
                            // Terminate tasks exceeding the maximum concurrency level.
                            if (_delegatesQueuedOrRunning > _maxDegreeOfParallelism)
                            {
                                _delegatesQueuedOrRunning--;
                                break;
                            }
                            // When there are no more items to be processed,
                            // note that we're done processing, and get out.
                            var node = _tasks.First;
                            if (node == null)
                            {
                                _delegatesQueuedOrRunning--;
                                break;
                            }
                            // Get the next item from the queue.
                            task = node.Value;
                            _tasks.RemoveFirst();
                        }

                        // If the task we pulled out is a long running task,
                        // executing the task on the pool thread.
                        if ((task.CreationOptions & TaskCreationOptions.LongRunning) != TaskCreationOptions.None)
                        {
                            RunTaskOnNewThread(task);
                            break;
                        }

                        // Execute the task we pulled out of the queue.
                        TryExecuteTask(task);
                    }
                }
                // We're done processing items on the current thread.
                finally
                {
                    _currentThreadIsProcessingItems = false;
                }
            }, null);
        }

        /// <summary>
        /// Run specified task on new (non pooled) thread.
        /// </summary>
        /// <param name="task">A task to execute.</param>
        private void RunTaskOnNewThread(Task task)
        {
            new Thread(param =>
            {
                // Note that the current thread is now processing work items.
                // This is necessary to enable inlining of tasks into this thread.
                _currentThreadIsProcessingItems = true;
                try
                {
                    TryExecuteTask((Task)param!);
                    while (true)
                    {
                        Task nextTask;
                        lock (_taskListLock)
                        {
                            if (_delegatesQueuedOrRunning > _maxDegreeOfParallelism)
                            {
                                _delegatesQueuedOrRunning--;
                                break;
                            }
                            // When there are no more items to be processed,
                            // note that we're done processing, and get out.
                            var node = _tasks.First;
                            if (node == null)
                            {
                                _delegatesQueuedOrRunning--;
                                break;
                            }
                            // Get the next item from the queue
                            nextTask = node.Value;

                            // If the task we pulled out is not a long running task and not allowed in non pooled thread,
                            // executing the task on the pool thread.
                            if (!_allowTORunNonLongRunningTaskOnNewThread
                                && (nextTask.CreationOptions & TaskCreationOptions.LongRunning) == TaskCreationOptions.None)
                            {
                                NotifyThreadPoolOfPendingWork();
                                break;
                            }

                            _tasks.RemoveFirst();
                        }

                        // Execute the task we pulled out of the queue.
                        TryExecuteTask(nextTask);
                    }
                }
                // We're done processing items on the current thread.
                finally
                {
                    _currentThreadIsProcessingItems = false;
                }
            })
            {
                IsBackground = true
            }.Start(task);
        }


#if !NET8_0_OR_GREATER
        /// <summary>
        /// Throw <see cref="ArgumentOutOfRangeException"/>.
        /// </summary>
        /// <typeparam name="T">The type of the objects.</typeparam>
        /// <param name="value">The maxDegreeOfParallelism of the argument that causes this exception.</param>
        /// <param name="other">The maxDegreeOfParallelism to compare with <paramref name="value"/>.</param>
        /// <param name="paramName">The name of the parameter with which <paramref name="value"/> corresponds.</param>
        /// <exception cref="ArgumentOutOfRangeException">Always thrown.</exception>
#if NETCOREAPP3_0_OR_GREATER || NETSTANDARD2_1
        [DoesNotReturn]
#endif  // NETCOREAPP3_0_OR_GREATER || NETSTANDARD2_1
        private static void ThrowLess<T>(T value, T other, string paramName)
        {
            throw new ArgumentOutOfRangeException(paramName, value, $"'{value}' must be greater than or equal to '{other}'.");
        }

        /// <summary>
        /// Throws an <see cref="ArgumentOutOfRangeException"/> if <paramref name="value"/> is less than <paramref name="other"/>.
        /// </summary>
        /// <typeparam name="T">The type of the objects to validate.</typeparam>
        /// <param name="value">The argument to validate as greater than or equal to <paramref name="other"/>.</param>
        /// <param name="other">The maxDegreeOfParallelism to compare with <paramref name="value"/>.</param>
        /// <param name="paramName">The name of the parameter with which <paramref name="value"/> corresponds.</param>
        /// <exception cref="ArgumentOutOfRangeException">Thrown if <paramref name="value"/> is less than <paramref name="other"/>.</exception>
        internal static void ThrowIfLessThan<T>(T value, T other, string paramName)
            where T : IComparable<T>
        {
            if (value.CompareTo(other) < 0)
            {
                ThrowLess(value, other, paramName);
            }
        }
#endif  // !NET8_0_OR_GREATER
    }
}
```

変更差分は追加メソッドを除けばそこまで大きくない．

```diff
diff --git LimitedConcurrencyLevelTaskScheduler.cs LimitedConcurrencyLevelTaskScheduler.cs
index 03dd972..eb1bbb2 100644
--- LimitedConcurrencyLevelTaskScheduler.cs
+++ LimitedConcurrencyLevelTaskScheduler.cs
@@ -46,13 +46,18 @@ namespace Koturn.Threading.Tasks
 #else
         private readonly object _taskListLock = new();
 #endif  // NET9_0_OR_GREATER
+        /// <summary>
+        /// True to allow non long running task in new (non pooled) thread.
+        /// </summary>
+        private readonly bool _allowTORunNonLongRunningTaskOnNewThread;


         /// <summary>
         /// Creates a new instance with the specified degree of parallelism.
         /// </summary>
         /// <param name="maxDegreeOfParallelism"></param>
-        public LimitedConcurrencyLevelTaskScheduler(int maxDegreeOfParallelism)
+        /// <param name="allowRunNonLongRunningTaskInNonPooledThread">True to allow non long running task in new (non pooled) thread.</param>
+        public LimitedConcurrencyLevelTaskScheduler(int maxDegreeOfParallelism, bool allowRunNonLongRunningTaskInNonPooledThread = true)
         {
 #if NET8_0_OR_GREATER
             ArgumentOutOfRangeException.ThrowIfLessThan(maxDegreeOfParallelism, 1);
@@ -60,6 +65,7 @@ namespace Koturn.Threading.Tasks
             ThrowIfLessThan(maxDegreeOfParallelism, 1, nameof(maxDegreeOfParallelism));
 #endif  // NET8_0_OR_GREATER
             _maxDegreeOfParallelism = maxDegreeOfParallelism;
+            _allowTORunNonLongRunningTaskOnNewThread = allowRunNonLongRunningTaskInNonPooledThread;
         }


@@ -103,15 +109,27 @@ namespace Koturn.Threading.Tasks
         /// <param name="task">A task.</param>
         protected sealed override void QueueTask(Task task)
         {
-            // Add the task to the list of tasks to be processed. If there aren't enough
-            // delegates currently queued or running to process tasks, schedule another.
             lock (_taskListLock)
             {
-                _tasks.AddLast(task);
                 if (_delegatesQueuedOrRunning < _maxDegreeOfParallelism)
                 {
                     _delegatesQueuedOrRunning++;
-                    NotifyThreadPoolOfPendingWork();
+                    if ((task.CreationOptions & TaskCreationOptions.LongRunning) == TaskCreationOptions.None)
+                    {
+                        // Add the task to the list of tasks to be processed.
+                        _tasks.AddLast(task);
+                        NotifyThreadPoolOfPendingWork();
+                    }
+                    else
+                    {
+                        // Run the long running task on the new thread.
+                        RunTaskOnNewThread(task);
+                    }
+                }
...skipping...
+        /// <summary>
+        /// Run specified task on new (non pooled) thread.
+        /// </summary>
+        /// <param name="task">A task to execute.</param>
+        private void RunTaskOnNewThread(Task task)
+        {
+            new Thread(param =>
+            {
+                // Note that the current thread is now processing work items.
+                // This is necessary to enable inlining of tasks into this thread.
+                _currentThreadIsProcessingItems = true;
+                try
+                {
+                    TryExecuteTask((Task)param!);
+                    while (true)
+                    {
+                        Task nextTask;
+                        lock (_taskListLock)
+                        {
+                            if (_delegatesQueuedOrRunning > _maxDegreeOfParallelism)
+                            {
+                                _delegatesQueuedOrRunning--;
+                                break;
+                            }
+                            // When there are no more items to be processed,
+                            // note that we're done processing, and get out.
+                            var node = _tasks.First;
+                            if (node == null)
+                            {
+                                _delegatesQueuedOrRunning--;
+                                break;
+                            }
+                            // Get the next item from the queue
+                            nextTask = node.Value;
+
+                            // If the task we pulled out is not a long running task and not allowed in non pooled thread,
+                            // executing the task on the pool thread.
+                            if (!_allowTORunNonLongRunningTaskOnNewThread
+                                && (nextTask.CreationOptions & TaskCreationOptions.LongRunning) == TaskCreationOptions.None)
+                            {
+                                NotifyThreadPoolOfPendingWork();
+                                break;
+                            }
+
+                            _tasks.RemoveFirst();
+                        }
+
+                        // Execute the task we pulled out of the queue.
+                        TryExecuteTask(nextTask);
+                    }
+                }
+                // We're done processing items on the current thread.
+                finally
+                {
+                    _currentThreadIsProcessingItems = false;
+                }
+            })
+            {
+                IsBackground = true
+            }.Start(task);
+        }
+

 #if !NET8_0_OR_GREATER
         /// <summary>
```

`QueueTask()` においては

- 現在の同時実行数 `_delegatesQueuedOrRunning` が許容同時実行数 `_maxDegreeOfParallelism` 未満かつ
    - 与えられた `Task` が `TaskCreationOptions.LongRunning` フラグを持つなら新規スレッドで実行
    - そうでなければスレッドプールのスレッドで実行
- 許容同時実行数以上ならペンディング（`_tasks` に追加）

とし，スレッドプールのスレッドで双方向リスト `_tasks` から取り取した `Task` が `TaskCreationOptions.LongRunning` フラグを持つなら，プールされたスレッドから新規スレッドを生成，そのスレッド上で実行するようにした．

生成したスレッドは指定された `Task` の処理を終えた後は，せっかく生成したスレッドなのだから，ということで，双方向リスト `_tasks` からタスクの取り出し・実行を行うようにしている．
ただし，その取り出したタスクが `TaskCreationOptions.LongRunning` フラグを持たないなら，どうしてもプールされたスレッドで実行したいということもあるかもしれないので， `_allowTORunNonLongRunningTaskOnNewThread` というフラグを設けている．
とりあえずコンストラクタでの指定のみとしているが，プロパティにして後から変更できるようにしても問題はないと思う．

### 簡単な動作確認

同時実行可能タスク数を2->4と4->2に変更する2例を確認する簡単な動作確認用プログラムを下記に示す．

タスクは開始と終了時にタイムスタンプ付きのログ出力と，その間に1000秒のスレッドのスリープを行うだけである．
元のスレッドではタスク開始の500ミリ秒後に同時実行数を変更するようにしている．

```cs
using System;
using System.Threading;
using System.Threading.Tasks;
using Koturn.Threading.Tasks;


namespace CSharpSandbox
{
    internal static class TaskSchedulerTest
    {
        public static void Main()
        {
            Test(2, 4);
            Test(4, 2);
        }

        public static void Test(int initialConcurrencyLevel, int concurrencyLevel)
        {
            Console.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}] Test started ===>");
            Console.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}] Initial concurrency level: {initialConcurrencyLevel}");

            var taskScheduler = new LimitedConcurrencyLevelTaskScheduler(initialConcurrencyLevel);
            var taskFactory = new TaskFactory(taskScheduler);

            var taskList = new Task[10];
            for (int i = 0; i < taskList.Length; i++)
            {
                var j = i;
                taskList[i] = taskFactory.StartNew(() =>
                {
                    var currentThread = Thread.CurrentThread;
                    Console.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}][{currentThread.ManagedThreadId}][{Task.CurrentId}][{currentThread.IsThreadPoolThread}] Task {j} Start");
                    Thread.Sleep(1000);
                    Console.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}][{currentThread.ManagedThreadId}][{Task.CurrentId}][{currentThread.IsThreadPoolThread}] Task {j} Finish");
                });
            }

            Thread.Sleep(500);

            Console.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}] Change concurrency level: {initialConcurrencyLevel} -> {concurrencyLevel}");
            taskScheduler.SetMaximumConcurrencyLevel(concurrencyLevel);

            Task.WaitAll(taskList);

            Console.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}] Test finished <===");
        }
    }
}
```

このプログラムの動作結果は以下のとおりで，

- 同時実行タスク数を2->4に増加させたとき，2タスクが設定直後に同時に実行開始
- 同時実行タスク数を4->2に減少させたとき，4タスクの終了後に立ち上がるのは2タスクのみ

という期待通りの動作となっていることが確認できる．

```
[2025-12-06 12:27:43.075] Test started ===>
[2025-12-06 12:27:43.076] Initial concurrency level: 2
[2025-12-06 12:27:43.097][4][1][True] Task 0 Start
[2025-12-06 12:27:43.098][3][2][True] Task 1 Start
[2025-12-06 12:27:43.602] Change concurrency level: 2 -> 4
[2025-12-06 12:27:43.603][6][3][True] Task 2 Start
[2025-12-06 12:27:43.604][5][4][True] Task 3 Start
[2025-12-06 12:27:44.099][3][2][True] Task 1 Finish
[2025-12-06 12:27:44.099][4][1][True] Task 0 Finish
[2025-12-06 12:27:44.100][3][5][True] Task 4 Start
[2025-12-06 12:27:44.101][4][6][True] Task 5 Start
[2025-12-06 12:27:44.612][6][3][True] Task 2 Finish
[2025-12-06 12:27:44.612][6][7][True] Task 6 Start
[2025-12-06 12:27:44.612][5][4][True] Task 3 Finish
[2025-12-06 12:27:44.613][5][8][True] Task 7 Start
[2025-12-06 12:27:45.112][3][5][True] Task 4 Finish
[2025-12-06 12:27:45.112][4][6][True] Task 5 Finish
[2025-12-06 12:27:45.112][4][10][True] Task 9 Start
[2025-12-06 12:27:45.112][3][9][True] Task 8 Start
[2025-12-06 12:27:45.624][6][7][True] Task 6 Finish
[2025-12-06 12:27:45.624][5][8][True] Task 7 Finish
[2025-12-06 12:27:46.116][3][9][True] Task 8 Finish
[2025-12-06 12:27:46.116][4][10][True] Task 9 Finish
[2025-12-06 12:27:46.116] Test finished <===
[2025-12-06 12:27:46.116] Test started ===>
[2025-12-06 12:27:46.116] Initial concurrency level: 4
[2025-12-06 12:27:46.116][6][11][True] Task 0 Start
[2025-12-06 12:27:46.117][5][14][True] Task 3 Start
[2025-12-06 12:27:46.117][3][13][True] Task 2 Start
[2025-12-06 12:27:46.116][7][12][True] Task 1 Start
[2025-12-06 12:27:46.622] Change concurrency level: 4 -> 2
[2025-12-06 12:27:47.119][6][11][True] Task 0 Finish
[2025-12-06 12:27:47.119][5][14][True] Task 3 Finish
[2025-12-06 12:27:47.150][3][13][True] Task 2 Finish
[2025-12-06 12:27:47.150][3][15][True] Task 4 Start
[2025-12-06 12:27:47.150][7][12][True] Task 1 Finish
[2025-12-06 12:27:47.152][7][16][True] Task 5 Start
[2025-12-06 12:27:48.157][7][16][True] Task 5 Finish
[2025-12-06 12:27:48.157][3][15][True] Task 4 Finish
[2025-12-06 12:27:48.158][3][18][True] Task 7 Start
[2025-12-06 12:27:48.157][7][17][True] Task 6 Start
[2025-12-06 12:27:49.164][3][18][True] Task 7 Finish
[2025-12-06 12:27:49.164][7][17][True] Task 6 Finish
[2025-12-06 12:27:49.165][7][20][True] Task 9 Start
[2025-12-06 12:27:49.164][3][19][True] Task 8 Start
[2025-12-06 12:27:50.174][7][20][True] Task 9 Finish
[2025-12-06 12:27:50.174][3][19][True] Task 8 Finish
[2025-12-06 12:27:50.174] Test finished <===
```

## 参考文献

- [TaskScheduler Class (System.Threading.Tasks) | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler "TaskScheduler Class (System.Threading.Tasks) | Microsoft Learn")
