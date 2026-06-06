# Filesystem — class hierarchy (UML)

UML-style Mermaid for the file, instance, watcher, and directory types across platforms. See
[`OVERVIEW.md`](OVERVIEW.md) and [`watchers.md`](watchers.md) for behavior.

---

## `File` family

```mermaid
classDiagram
    class File {
        <<interface>>
        +open(FlagSet)* IoCallBoolResult
        +write(buffer)* IoCallSizeResult
        +pread(buf, count, offset)* IoCallSizeResult
        +pwrite(buf, count, offset)* IoCallSizeResult
        +info()* IoCallResult~FileInfo~
        +close()* IoCallBoolResult
        +isOpen()* bool
        +path()* string_view
        +destinationType()* DestinationType
    }

    class FileSharedImpl {
        #fd_ : filesystem_os_id_t
        #filepath_and_type_ : FilePathAndType
        +isOpen() / path() / destinationType()
        #generateTmpFilePath(path)$
    }
    class FileImplPosix {
        +open/write/close/pread/pwrite/info
        #translateFlag(FlagSet) FlagsAndMode
    }
    class TmpFileImplPosix {
        -tmp_file_path_
        -openNamedTmpFile(flags, with_unlink)
    }
    class FileImplWin32

    File <|.. FileSharedImpl
    FileSharedImpl <|-- FileImplPosix
    FileImplPosix <|-- TmpFileImplPosix
    FileSharedImpl <|-- FileImplWin32

    note for FileSharedImpl "cross-platform state + accessors"
    note for TmpFileImplPosix "nameless tmp (O_TMPFILE)\nfallback: named + unlink"
```

---

## `Instance` family + value types

```mermaid
classDiagram
    class Instance {
        <<interface>>
        +createFile(FilePathAndType)* FilePtr
        +fileExists(path)* bool
        +directoryExists(path)* bool
        +stat(path)* IoCallResult~FileInfo~
        +createPath(path)* IoCallBoolResult
        +fileSize(path)* ssize_t
        +fileReadToEnd(path)* StatusOr~string~
        +splitPathFromFilename(path)* StatusOr~PathSplitResult~
        +illegalPath(path)* bool
    }
    class InstanceImplPosix {
        -canonicalPath(path)
    }
    class InstanceImplWin32

    Instance <|.. InstanceImplPosix
    Instance <|.. InstanceImplWin32

    class FileInfo {
        +name_ / size_ / file_type_
        +time_created_ / time_last_accessed_ / time_last_modified_
    }
    class FilePathAndType {
        +file_type_ : DestinationType
        +path_ : string
    }
    class PathSplitResult {
        +directory_ / file_
    }
    Instance ..> FileInfo
    Instance ..> FilePathAndType
    Instance ..> PathSplitResult
```

---

## Error type

```mermaid
classDiagram
    class IoError { <<interface>> }
    class IoFileError {
        -errno_ : int
        +getErrorCode() IoErrorCode
        +getErrorDetails() string
        +getSystemErrorCode() int
    }
    IoError <|.. IoFileError
    note for IoFileError "wraps raw errno\nbuilt via resultSuccess/resultFailure"
```

---

## `Watcher` family

```mermaid
classDiagram
    class Watcher {
        <<interface>>
        +addWatch(path, events, cb)* Status
    }
    class WatcherImpl_inotify {
        -inotify_fd_ : int
        -inotify_event_ : FileEventPtr
        -callback_map_ : map~int, DirectoryWatch~
        -onInotifyEvent() Status
    }
    class WatcherImpl_kqueue {
        -queue_ : int
        -kqueue_event_ : FileEventPtr
        -watches_ : map~int, FileWatchPtr~
        -onKqueueEvent() Status
        -addWatch(path, events, cb, pathMustExist)
        -removeWatch(watch)
    }
    class WatcherImpl_win32

    Watcher <|.. WatcherImpl_inotify
    Watcher <|.. WatcherImpl_kqueue
    Watcher <|.. WatcherImpl_win32

    class FileWatch_inotify {
        +file_ / events_ / cb_
    }
    class DirectoryWatch {
        +watches_ : list~FileWatch~
    }
    class FileWatch_kqueue {
        +fd_ / events_ / file_ / callback_ / watching_dir_
        +~FileWatch() close(fd_)
    }

    WatcherImpl_inotify *-- DirectoryWatch
    DirectoryWatch *-- FileWatch_inotify
    WatcherImpl_kqueue *-- FileWatch_kqueue

    note for WatcherImpl_inotify "watches the directory by name;\nfilters by filename"
    note for WatcherImpl_kqueue "watches an fd;\nre-attaches on rename/delete"
```

---

## Directory iteration

```mermaid
classDiagram
    class DirectoryIterator {
        #entry_ : DirectoryEntry
        #status_ : Status
        +operator*() DirectoryEntry&
        +operator!=(rhs) bool
        +operator++()* DirectoryIteratorImpl&
        +status() Status&
    }
    class DirectoryIteratorImpl {
        -directory_path_ / dir_ : DIR* / os_sys_calls_
        -nextEntry() / openDirectory()
        -makeEntry(filename) StatusOr~DirectoryEntry~
    }
    class Directory {
        -directory_path_
        +begin() DirectoryIteratorImpl
        +end() DirectoryIteratorImpl
    }
    class DirectoryEntry {
        +name_ / type_ / size_bytes_
    }

    DirectoryIterator <|-- DirectoryIteratorImpl
    Directory ..> DirectoryIteratorImpl : begin()/end()
    DirectoryIterator ..> DirectoryEntry

    note for DirectoryIteratorImpl "move-only (DIR* not shareable)\nfails silently; check status()"
```

---

## Type alias reference

| Alias | Underlying | Meaning |
|---|---|---|
| `FilePtr` | `unique_ptr<File>` | Owning file handle. |
| `InstancePtr` | `unique_ptr<Instance>` | Owning filesystem facade. |
| `WatcherPtr` | `unique_ptr<Watcher>` | Owning watcher. |
| `FlagSet` | `bitset<5>` | Open-operation flags. |
| `IoFileErrorPtr` | `unique_ptr<IoFileError, deleter>` | Error with custom deleter. |
| `FileImpl` / `InstanceImpl` | platform alias | `FileImplPosix` / `InstanceImplPosix` on POSIX. |
| `FileWatchPtr` (kqueue) | `shared_ptr<FileWatch>` | One watch entry; dtor closes fd. |

---

## Relationship summary

| Relationship | Type | Meaning |
|---|---|---|
| `FileImplPosix` → `FileSharedImpl` → `File` | inheritance | Cross-platform base + POSIX syscalls. |
| `TmpFileImplPosix` → `FileImplPosix` | inheritance | Temp-file specialization. |
| `WatcherImpl` → `Watcher` | inheritance | One per platform (inotify/kqueue/win32). |
| `WatcherImpl` → `FileEvent` | composition | Notification fd on the dispatcher. |
| `WatcherImpl` → `Filesystem::Instance` | uses | `splitPathFromFilename`. |
| `Directory` → `DirectoryIteratorImpl` | factory | `begin()`/`end()` for range-for. |
| `IoFileError` → `IoError` | inheritance | errno-carrying error. |
