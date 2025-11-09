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

```cs
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

#if !NET9_0_OR_GREATER
using Lock = object;
#endif  // !NET9_0_OR_GREATER


namespace Koturn.Tasks
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
        private readonly Lock _taskListLock = new();


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
                        Task item;
                        lock (_taskListLock)
                        {
                            // When there are no more items to be processed,
                            // note that we're done processing, and get out.
                            var task = _tasks.First;
                            if (task == null)
                            {
                                _delegatesQueuedOrRunning--;
                                break;
                            }
                            // Get the next item from the queue.
                            item = task.Value;
                            _tasks.RemoveFirst();
                        }
                        // Execute the task we pulled out of the queue.
                        TryExecuteTask(item);
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

```cs
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

#if !NET9_0_OR_GREATER
using Lock = object;
#endif  // !NET9_0_OR_GREATER


namespace Koturn.Tasks
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
        private readonly Lock _taskListLock = new();


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
                        Task item;
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
                            var task = _tasks.First;
                            if (task == null)
                            {
                                _delegatesQueuedOrRunning--;
                                break;
                            }
                            // Get the next item from the queue.
                            item = task.Value;
                            _tasks.RemoveFirst();
                        }
                        // Execute the task we pulled out of the queue.
                        TryExecuteTask(item);
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
@@ -63,6 +63,40 @@ namespace Koturn.Tasks
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
@@ -174,6 +208,12 @@ namespace Koturn.Tasks
                         Task item;
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
                             var task = _tasks.First;
```

`MaximumConcurrencyLevel` は親クラスの設計に依存し，getのみのプロパティであるため，別途設定用のメソッドである `SetMaximumConcurrencyLevel()` を設けた．
同時実行数を増やした場合，ペンディングとなっているタスクを即座に実行するようにしてある．

反対に同時実行数を減らした場合は， `ThreadPool.UnsafeQueueUserWorkItem()` に与えるラムダ式にて，次の実行タスクを取り出す前に，現在のタスク数と設定されている同時実行数を比較し，超過していればそのスレッドを終了する処理を加えている．
現在実行中のTaskを停止させるのではなく，次のタスクの取り出しの際にスレッド数を減らす作りのため，同時実行数を減らす場合の反映はラグがある．

### 簡単な動作確認

同時実行可能タスク数を2->4と4->2に変更する2例を確認する簡単な動作確認用プログラムを下記に示す．

タスクは開始と終了時にタイムスタンプ付きのログ出力と，その間に1000秒のスレッドのスリープを行うだけである．
元のスレッドではタスク開始の500ミリ秒後に同時実行数を変更するようにしている．

```cs
using System;
using System.Threading;
using System.Threading.Tasks;
using Koturn.Tasks;


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
                    Console.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}] Task {j} Start");
                    Thread.Sleep(1000);
                    Console.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}] Task {j} Finish");
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
[2025-11-09 12:17:53.436] Test started ===>
[2025-11-09 12:17:53.438] Initial concurrency level: 2
[2025-11-09 12:17:53.461] Task 0 Start
[2025-11-09 12:17:53.461] Task 1 Start
[2025-11-09 12:17:53.973] Change concurrency level: 2 -> 4
[2025-11-09 12:17:53.973] Task 2 Start
[2025-11-09 12:17:53.975] Task 3 Start
[2025-11-09 12:17:54.471] Task 1 Finish
[2025-11-09 12:17:54.471] Task 0 Finish
[2025-11-09 12:17:54.472] Task 4 Start
[2025-11-09 12:17:54.472] Task 5 Start
[2025-11-09 12:17:54.982] Task 2 Finish
[2025-11-09 12:17:54.982] Task 6 Start
[2025-11-09 12:17:54.982] Task 3 Finish
[2025-11-09 12:17:54.983] Task 7 Start
[2025-11-09 12:17:55.479] Task 5 Finish
[2025-11-09 12:17:55.479] Task 4 Finish
[2025-11-09 12:17:55.479] Task 8 Start
[2025-11-09 12:17:55.479] Task 9 Start
[2025-11-09 12:17:55.987] Task 7 Finish
[2025-11-09 12:17:55.987] Task 6 Finish
[2025-11-09 12:17:56.486] Task 8 Finish
[2025-11-09 12:17:56.486] Task 9 Finish
[2025-11-09 12:17:56.487] Test finished <===
[2025-11-09 12:17:56.487] Test started ===>
[2025-11-09 12:17:56.487] Initial concurrency level: 4
[2025-11-09 12:17:56.487] Task 0 Start
[2025-11-09 12:17:56.487] Task 3 Start
[2025-11-09 12:17:56.487] Task 1 Start
[2025-11-09 12:17:56.487] Task 2 Start
[2025-11-09 12:17:56.995] Change concurrency level: 4 -> 2
[2025-11-09 12:17:57.492] Task 3 Finish
[2025-11-09 12:17:57.492] Task 1 Finish
[2025-11-09 12:17:57.492] Task 2 Finish
[2025-11-09 12:17:57.493] Task 4 Start
[2025-11-09 12:17:57.492] Task 0 Finish
[2025-11-09 12:17:57.495] Task 5 Start
[2025-11-09 12:17:58.499] Task 4 Finish
[2025-11-09 12:17:58.499] Task 5 Finish
[2025-11-09 12:17:58.501] Task 7 Start
[2025-11-09 12:17:58.500] Task 6 Start
[2025-11-09 12:17:59.518] Task 7 Finish
[2025-11-09 12:17:59.518] Task 6 Finish
[2025-11-09 12:17:59.519] Task 9 Start
[2025-11-09 12:17:59.518] Task 8 Start
[2025-11-09 12:18:00.533] Task 8 Finish
[2025-11-09 12:18:00.533] Task 9 Finish
[2025-11-09 12:18:00.535] Test finished <===
```

## 参考文献

- [TaskScheduler Class (System.Threading.Tasks) | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler "TaskScheduler Class (System.Threading.Tasks) | Microsoft Learn")
