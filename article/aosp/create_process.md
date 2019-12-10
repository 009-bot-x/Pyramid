> 本文基于 Android 9.0 , 代码仓库地址 ： [android_9.0.0_r45](https://github.com/lulululbj/android_9.0.0_r45)
>
> 文中源码链接：
>
> [SystemServer.java](https://github.com/lulululbj/android_9.0.0_r45/blob/master/frameworks/base/services/java/com/android/server/SystemServer.java)
>
> [ActivityManagerService.java](https://github.com/lulululbj/android_9.0.0_r45/blob/master/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java)
>
> [Process.java](https://github.com/lulululbj/android_9.0.0_r45/blob/master/frameworks/base/core/java/android/os/Process.java)
>
> [ZygoteProcess.java](https://github.com/lulululbj/android_9.0.0_r45/blob/master/frameworks/base/core/java/android/os/ZygoteProcess.java)

对 `Zygote` 和 `SystemServer` 启动流程还不熟悉的建议阅读下面两篇文章：

> [Java 世界的盘古和女娲 —— Zygote](https://juejin.im/post/5d8f73bf51882555b149dc64)
>
> [Zygote 家的大儿子 —— SystemServer](https://juejin.im/post/5da341f451882561ba64b9da)

`Zygote` 作为 Android 世界的受精卵，在成功繁殖出 `system_server` 进程之后并没有完全功成身退，仍然承担着受精卵的责任。`Zygote` 通过调用其持有的 `ZygoteServer` 对象的 `runSelectLoop()` 方法开始等待客户端的呼唤，有求必应。客户端的请求无非是创建应用进程，以 `startActivity()` 为例，假如开启的是一个尚未创建进程的应用，那么就会向 Zygote 请求创建进程。下面将从 **客户端发送请求** 和 **服务端处理请求** 两方面来进行解析。

## 客户端发送请求

`startActivity()` 的具体流程这里就不分析了，系列后续文章会写到。我们直接看到创建进程的 `startProcess()` 方法，该方法在 `ActivityManagerService` 中，后面简称 `AMS`。

### Process.startProcess()

```java
> ActivityManagerService.java

private ProcessStartResult startProcess(String hostingType, String entryPoint,
        ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
        String seInfo, String requiredAbi, String instructionSet, String invokeWith,
        long startTime) {
    try {
        checkTime(startTime, "startProcess: asking zygote to start proc");
        final ProcessStartResult startResult;
        if (hostingType.equals("webview_service")) {
            startResult = startWebView(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, null,
                    new String[] {PROC_START_SEQ_IDENT + app.startSeq});
        } else {
            // 新建进程
            startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, invokeWith,
                    new String[] {PROC_START_SEQ_IDENT + app.startSeq});
        }
        checkTime(startTime, "startProcess: returned from zygote!");
        return startResult;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```

调用 `Process.start()` 方法新建进程，继续追进去:

```java
> Process.java

public static final ProcessStartResult start(
                // android.app.ActivityThread，创建进程后会调用其 main() 方法
                final String processClass,
                final String niceName, // 进程名
                int uid, int gid, int[] gids,
                int runtimeFlags, int mountExternal,
                int targetSdkVersion,
                String seInfo,
                String abi,
                String instructionSet,
                String appDataDir,
                String invokeWith, // 一般新建应用进程时，此参数不为 null
                String[] zygoteArgs) {
        return zygoteProcess.start(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
    }
```

继续调用 `zygoteProcess.start()` ：

```java
> ZygoteProess.java

public final Process.ProcessStartResult start(final String processClass,
                                              final String niceName,
                                              int uid, int gid, int[] gids,
                                              int runtimeFlags, int mountExternal,
                                              int targetSdkVersion,
                                              String seInfo,
                                              String abi,
                                              String instructionSet,
                                              String appDataDir,
                                              String invokeWith,
                                              String[] zygoteArgs) {
    try {
        return startViaZygote(processClass, niceName, uid, gid, gids,
                runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, false /* startChildZygote */,
                zygoteArgs);
    } catch (ZygoteStartFailedEx ex) {
        Log.e(LOG_TAG,
                "Starting VM process through Zygote failed");
        throw new RuntimeException(
                "Starting VM process through Zygote failed", ex);
    }
}
```

调用 `startViaZygote()` 方法。终于看到 Zygote 的身影了。

### startViaZygote()

```java
> ZygoteProcess.java

private Process.ProcessStartResult startViaZygote(final String processClass,
                                                  final String niceName,
                                                  final int uid, final int gid,
                                                  final int[] gids,
                                                  int runtimeFlags, int mountExternal,
                                                  int targetSdkVersion,
                                                  String seInfo,
                                                  String abi,
                                                  String instructionSet,
                                                  String appDataDir,
                                                  String invokeWith,
                                                  boolean startChildZygote, // 是否克隆 zygote 进程的所有状态
                                                  String[] extraArgs)
                                                  throws ZygoteStartFailedEx {
    ArrayList<String> argsForZygote = new ArrayList<String>();

    // --runtime-args, --setuid=, --setgid=,
    // and --setgroups= must go first
    // 处理参数
    argsForZygote.add("--runtime-args");
    argsForZygote.add("--setuid=" + uid);
    argsForZygote.add("--setgid=" + gid);
    argsForZygote.add("--runtime-flags=" + runtimeFlags);
    if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
        argsForZygote.add("--mount-external-default");
    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
        argsForZygote.add("--mount-external-read");
    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
        argsForZygote.add("--mount-external-write");
    }
    argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

    // --setgroups is a comma-separated list
    if (gids != null && gids.length > 0) {
        StringBuilder sb = new StringBuilder();
        sb.append("--setgroups=");

        int sz = gids.length;
        for (int i = 0; i < sz; i++) {
            if (i != 0) {
                sb.append(',');
            }
            sb.append(gids[i]);
        }

        argsForZygote.add(sb.toString());
    }

    if (niceName != null) {
        argsForZygote.add("--nice-name=" + niceName);
    }

    if (seInfo != null) {
        argsForZygote.add("--seinfo=" + seInfo);
    }

    if (instructionSet != null) {
        argsForZygote.add("--instruction-set=" + instructionSet);
    }

    if (appDataDir != null) {
        argsForZygote.add("--app-data-dir=" + appDataDir);
    }

    if (invokeWith != null) {
        argsForZygote.add("--invoke-with");
        argsForZygote.add(invokeWith);
    }

    if (startChildZygote) {
        argsForZygote.add("--start-child-zygote");
    }

    argsForZygote.add(processClass);

    if (extraArgs != null) {
        for (String arg : extraArgs) {
            argsForZygote.add(arg);
        }
    }

    synchronized(mLock) {
        // 和 Zygote 进程进行 socket 通信
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
    }
}
```

前面一大串代码都是在处理参数，大致浏览即可。核心在于最后的 `openZygoteSocketIfNeeded()` 和 `zygoteSendArgsAndGetResult()` 这两个方法。从方法命名就可以看出来，这里要和 `Zygote` 进行 socket 通信了。还记得 `ZygoteInit.main()` 方法中调用的 `registerServerSocketFromEnv()` 方法吗？它在 Zygote 进程中创建了服务端 socket。

### openZygoteSocketIfNeeded()

先来看看 `openZygoteSocketIfNeeded()` 方法。

```java
> ZygoteProcess.java

private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
    Preconditions.checkState(Thread.holdsLock(mLock), "ZygoteProcess lock not held");

    // 未连接或者连接已关闭
    if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
        try {
            // 开启 socket 连接
            primaryZygoteState = ZygoteState.connect(mSocket);
        } catch (IOException ioe) {
            throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
        }
        maybeSetApiBlacklistExemptions(primaryZygoteState, false);
        maybeSetHiddenApiAccessLogSampleRate(primaryZygoteState);
    }
    if (primaryZygoteState.matches(abi)) {
        return primaryZygoteState;
    }

    // 当主 zygote 没有匹配成功，尝试 connect 第二个 zygote
    if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
        try {
            secondaryZygoteState = ZygoteState.connect(mSecondarySocket);
        } catch (IOException ioe) {
            throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
        }
        maybeSetApiBlacklistExemptions(secondaryZygoteState, false);
        maybeSetHiddenApiAccessLogSampleRate(secondaryZygoteState);
    }

    if (secondaryZygoteState.matches(abi)) {
        return secondaryZygoteState;
    }

    throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
}
```

 如果与 Zygote 进程的 socket 连接未开启，则尝试开启，可能会产生阻塞和重试。连接调用的是 `ZygoteState.connect()` 方法，`ZygoteState` 是 `ZygoteProcess` 的内部类。

 ```java
 > ZygoteProcess.java

 public static class ZygoteState {
        final LocalSocket socket;
        final DataInputStream inputStream;
        final BufferedWriter writer;
        final List<String> abiList;

        boolean mClosed;

        private ZygoteState(LocalSocket socket, DataInputStream inputStream,
                BufferedWriter writer, List<String> abiList) {
            this.socket = socket;
            this.inputStream = inputStream;
            this.writer = writer;
            this.abiList = abiList;
        }

        public static ZygoteState connect(LocalSocketAddress address) throws IOException {
            DataInputStream zygoteInputStream = null;
            BufferedWriter zygoteWriter = null;
            final LocalSocket zygoteSocket = new LocalSocket();

            try {
                zygoteSocket.connect(address);

                zygoteInputStream = new DataInputStream(zygoteSocket.getInputStream());

                zygoteWriter = new BufferedWriter(new OutputStreamWriter(
                        zygoteSocket.getOutputStream()), 256);
            } catch (IOException ex) {
                try {
                    zygoteSocket.close();
                } catch (IOException ignore) {
                }

                throw ex;
            }

            String abiListString = getAbiList(zygoteWriter, zygoteInputStream);
            Log.i("Zygote", "Process: zygote socket " + address.getNamespace() + "/"
                    + address.getName() + " opened, supported ABIS: " + abiListString);

            return new ZygoteState(zygoteSocket, zygoteInputStream, zygoteWriter,
                    Arrays.asList(abiListString.split(",")));
        }
    ...
}
 ```

 通过 socket 连接 Zygote 远程服务端。

 再回头看之前的 `zygoteSendArgsAndGetResult()` 方法。

### zygoteSendArgsAndGetResult()

 ```java
  > ZygoteProcess.java

 private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
        ZygoteState zygoteState, ArrayList<String> args)
        throws ZygoteStartFailedEx {
    try {
        ...
        final BufferedWriter writer = zygoteState.writer;
        final DataInputStream inputStream = zygoteState.inputStream;

        writer.write(Integer.toString(args.size()));
        writer.newLine();

        // 向 zygote 进程发送参数
        for (int i = 0; i < sz; i++) {
            String arg = args.get(i);
            writer.write(arg);
            writer.newLine();
        }

        writer.flush();

        // 是不是应该有一个超时时间？
        Process.ProcessStartResult result = new Process.ProcessStartResult();

        // Always read the entire result from the input stream to avoid leaving
        // bytes in the stream for future process starts to accidentally stumble
        // upon.
        // 读取 zygote 进程返回的子进程 pid
        result.pid = inputStream.readInt();
        result.usingWrapper = inputStream.readBoolean();

        if (result.pid < 0) { // pid 小于 0 ，fork 失败
            throw new ZygoteStartFailedEx("fork() failed");
        }
        return result;
    } catch (IOException ex) {
        zygoteState.close();
        throw new ZygoteStartFailedEx(ex);
    }
}
 ```

 通过 socket 发送请求参数，然后等待 Zygote 进程返回子进程 pid 。客户端的工作到这里就暂时完成了，我们再追踪到服务端，看看服务端是如何处理客户端请求的。

## Zygote 处理客户端请求

`Zygote` 处理客户端请求的代码在 `ZygoteServer.runSelectLoop()` 方法中。

 ```java
 > ZygoteServer.java

 Runnable runSelectLoop(String abiList) {
    ...

    while (true) {
       ...
        try {
            // 有事件来时往下执行，没有时就阻塞
            Os.poll(pollFds, -1);
        } catch (ErrnoException ex) {
            throw new RuntimeException("poll failed", ex);
        }
        for (int i = pollFds.length - 1; i >= 0; --i) {
            if ((pollFds[i].revents & POLLIN) == 0) {
                continue;
            }

            if (i == 0) { // 有新客户端连接
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else { // 处理客户端请求
                try {
                    ZygoteConnection connection = peers.get(i);
                    // fork 子进程，并返回包含子进程 main() 函数的 Runnable 对象
                    final Runnable command = connection.processOneCommand(this);

                    if (mIsForkChild) {
                        // 位于子进程
                        if (command == null) {
                            throw new IllegalStateException("command == null");
                        }

                        return command;
                    } else {
                        // 位于父进程
                        if (command != null) {
                            throw new IllegalStateException("command != null");
                        }

                        if (connection.isClosedByPeer()) {
                            connection.closeSocket();
                            peers.remove(i);
                            fds.remove(i);
                        }
                    }
                } catch (Exception e) {
                    ...
                } finally {
                    mIsForkChild = false;
                }
            }
        }
    }
}
```

`acceptCommandPeer()` 方法用来响应新客户端的 socket 连接请求。`processOneCommand()` 方法用来处理客户端的一般请求。

### processOneCommand()

```java
> ZygoteConnection.java

Runnable processOneCommand(ZygoteServer zygoteServer) {
    String args[];
    Arguments parsedArgs = null;
    FileDescriptor[] descriptors;

    try {
        // 1. 读取 socket 客户端发送过来的参数列表
        args = readArgumentList();
        descriptors = mSocket.getAncillaryFileDescriptors();
    } catch (IOException ex) {
        throw new IllegalStateException("IOException on command socket", ex);
    }

    ...

    // 2. fork 子进程
    pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
            parsedArgs.runtimeFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
            parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.startChildZygote,
            parsedArgs.instructionSet, parsedArgs.appDataDir);

    try {
        if (pid == 0) {
            // 处于进子进程
            zygoteServer.setForkChild();
            // 关闭服务端 socket
            zygoteServer.closeServerSocket();
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;
            // 3. 处理子进程事务
            return handleChildProc(parsedArgs, descriptors, childPipeFd,
                    parsedArgs.startChildZygote);
        } else {
            // 处于 Zygote 进程
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            // 4. 处理父进程事务
            handleParentProc(pid, descriptors, serverPipeFd);
            return null;
        }
    } finally {
        IoUtils.closeQuietly(childPipeFd);
        IoUtils.closeQuietly(serverPipeFd);
    }
}
```

`processOneCommand()` 方法大致可以分为五步，下面逐步分析。

#### readArgumentList()

```java
> ZygoteConnection.java

private String[] readArgumentList()
        throws IOException {

    int argc;

    try {
        // 逐行读取参数
        String s = mSocketReader.readLine();

        if (s == null) {
            // EOF reached.
            return null;
        }
        argc = Integer.parseInt(s);
    } catch (NumberFormatException ex) {
        throw new IOException("invalid wire format");
    }

    // See bug 1092107: large argc can be used for a DOS attack
    if (argc > MAX_ZYGOTE_ARGC) {
        throw new IOException("max arg count exceeded");
    }

    String[] result = new String[argc];
    for (int i = 0; i < argc; i++) {
        result[i] = mSocketReader.readLine();
        if (result[i] == null) {
            // We got an unexpected EOF.
            throw new IOException("truncated request");
        }
    }

    return result;
}
```

读取客户端发送过来的请求参数。

#### forkAndSpecialize()

```java
> Zygote.java

public static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags,
      int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
      int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir) {
    VM_HOOKS.preFork();
    // Resets nice priority for zygote process.
    resetNicePriority();
    int pid = nativeForkAndSpecialize(
              uid, gid, gids, runtimeFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
              fdsToIgnore, startChildZygote, instructionSet, appDataDir);
    // Enable tracing as soon as possible for the child process.
    if (pid == 0) {
        Trace.setTracingEnabled(true, runtimeFlags);

        // Note that this event ends at the end of handleChildProc,
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "PostFork");
    }
    VM_HOOKS.postForkCommon();
    return pid;
}
```

`nativeForkAndSpecialize()` 是一个 native 方法，在底层 fork 了一个新进程，并返回其 pid。不要忘记了这里的 **一次fork，两次返回** 。`pid > 0`  说明还是父进程。`pid = 0` 说明进入了子进程。子进程中会调用 `handleChildProc`，而父进程中会调用 `handleParentProc()`。

#### handleChildProc()
```java
> ZygoteConnection.java

private Runnable handleChildProc(Arguments parsedArgs, FileDescriptor[] descriptors,
        FileDescriptor pipeFd, boolean isZygote) {
    closeSocket(); // 关闭 socket 连接
    ...

    if (parsedArgs.niceName != null) {
        // 设置进程名
        Process.setArgV0(parsedArgs.niceName);
    }

    if (parsedArgs.invokeWith != null) {
        WrapperInit.execApplication(parsedArgs.invokeWith,
                parsedArgs.niceName, parsedArgs.targetSdkVersion,
                VMRuntime.getCurrentInstructionSet(),
                pipeFd, parsedArgs.remainingArgs);

        // Should not get here.
        throw new IllegalStateException("WrapperInit.execApplication unexpectedly returned");
    } else {
        if (!isZygote) { // 新建应用进程时 isZygote 参数为 false
            return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs,
                    null /* classLoader */);
        } else {
            return ZygoteInit.childZygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null /* classLoader */);
        }
    }
}
```

当看到 `ZygoteInit.zygoteInit()` 时你应该感觉很熟悉了，接下来的流程就是：

> `ZygoteInit.zygoteInit()` -> `RuntimeInit.applicationInit()` -> `findStaticMain()`

和 `SystemServer` 进程的创建流程一致。这里要找的 main 方法就是 `ActivityThrad.main()` 。`ActivityThread` 虽然并不是一个线程，但你可以把它理解为应用的主线程。

### handleParentProc()

```java
> ZygoteConnection.java

private void handleParentProc(int pid, FileDescriptor[] descriptors, FileDescriptor pipeFd) {
        if (pid > 0) {
            setChildPgid(pid);
        }

        if (descriptors != null) {
            for (FileDescriptor fd: descriptors) {
                IoUtils.closeQuietly(fd);
            }
        }

        boolean usingWrapper = false;
        if (pipeFd != null && pid > 0) {
            int innerPid = -1;
            try {
                // Do a busy loop here. We can't guarantee that a failure (and thus an exception
                // bail) happens in a timely manner.
                final int BYTES_REQUIRED = 4;  // Bytes in an int.

                StructPollfd fds[] = new StructPollfd[] {
                        new StructPollfd()
                };

                byte data[] = new byte[BYTES_REQUIRED];

                int remainingSleepTime = WRAPPED_PID_TIMEOUT_MILLIS;
                int dataIndex = 0;
                long startTime = System.nanoTime();

                while (dataIndex < data.length && remainingSleepTime > 0) {
                    fds[0].fd = pipeFd;
                    fds[0].events = (short) POLLIN;
                    fds[0].revents = 0;
                    fds[0].userData = null;

                    int res = android.system.Os.poll(fds, remainingSleepTime);
                    long endTime = System.nanoTime();
                    int elapsedTimeMs = (int)((endTime - startTime) / 1000000l);
                    remainingSleepTime = WRAPPED_PID_TIMEOUT_MILLIS - elapsedTimeMs;

                    if (res > 0) {
                        if ((fds[0].revents & POLLIN) != 0) {
                            // Only read one byte, so as not to block.
                            int readBytes = android.system.Os.read(pipeFd, data, dataIndex, 1);
                            if (readBytes < 0) {
                                throw new RuntimeException("Some error");
                            }
                            dataIndex += readBytes;
                        } else {
                            // Error case. revents should contain one of the error bits.
                            break;
                        }
                    } else if (res == 0) {
                        Log.w(TAG, "Timed out waiting for child.");
                    }
                }

                if (dataIndex == data.length) {
                    DataInputStream is = new DataInputStream(new ByteArrayInputStream(data));
                    innerPid = is.readInt();
                }

                if (innerPid == -1) {
                    Log.w(TAG, "Error reading pid from wrapped process, child may have died");
                }
            } catch (Exception ex) {
                Log.w(TAG, "Error reading pid from wrapped process, child may have died", ex);
            }

            // Ensure that the pid reported by the wrapped process is either the
            // child process that we forked, or a descendant of it.
            if (innerPid > 0) {
                int parentPid = innerPid;
                while (parentPid > 0 && parentPid != pid) {
                    parentPid = Process.getParentPid(parentPid);
                }
                if (parentPid > 0) {
                    Log.i(TAG, "Wrapped process has pid " + innerPid);
                    pid = innerPid;
                    usingWrapper = true;
                } else {
                    Log.w(TAG, "Wrapped process reported a pid that is not a child of "
                            + "the process that we forked: childPid=" + pid
                            + " innerPid=" + innerPid);
                }
            }
        }

        try {
            mSocketOutStream.writeInt(pid);
            mSocketOutStream.writeBoolean(usingWrapper);
        } catch (IOException ex) {
            throw new IllegalStateException("Error writing to command socket", ex);
        }
    }
```

主要进行一些资源清理的工作。到这里，子进程就创建完成了。

## 总结


![](https://user-gold-cdn.xitu.io/2019/10/15/16dcffa1150587ce?w=1211&h=762&f=png&s=72152)

1. 调用 `Process.start()` 创建应用进程
2. `ZygoteProcess` 负责和 `Zygote` 进程建立 socket 连接，并将创建进程需要的参数发送给 Zygote 的 socket 服务端
3. `Zygote` 服务端接收到参数之后调用 `ZygoteConnection.processOneCommand()` 处理参数，并 fork 进程
4. 最后通过 `findStaticMain()` 找到 `ActivityThread` 类的 main() 方法并执行，子进程就启动了

## 预告

到现在为止已经解析了 `Zygote` 进程 ，`SystemServer` 进程，以及应用进程的创建。下一篇的内容是和应用最密切相关的系统服务 **ActivityManagerService** , 来看看它在 `SystemServer` 中是如何被创建和启动的，敬请期待！

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多最新原创文章，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
