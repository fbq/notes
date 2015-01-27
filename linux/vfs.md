# Namespace
In Unix, filesystems are mounted at a specific mount point in a global hierarchy known as a namespace.

Recently, Linux has made this hierarchy per-process, to give a unique namespace to each process.
Because each process inherits its parent’s namespace (unless you specify otherwise), there is seemingly
one global namespace.

# VFS Objects
* superblock, represents a specific mounted filesystem.
* inode, represents a specific file.
* dentry, represents a directory entry, which is a single component of a path.
* file, represents an open file as associated with a process.
* operations, contained within each of these primary objects.
  These objects describe the methods that the kernel invokes against the primary objects

## *superblock* object

All functions in `super_operations` are invoked by the VFS, in process context. All except
`dirty_inode()` may all block if needed.

## *inode* object

An inode represents each file on a filesystem, but the inode object is constructed in
memory only as files are accessed.

## *dentry* object

Dentry objects are all components in a path, including files, and don't correspond to any
sort of on-disk data structure.

### dentry state

|status  |condition                           |meaning                                                            |
|--------|------------------------------------|-------------------------------------------------------------------|
|used    |`d_inode` != NULL && `d_count` > 0  |the dentry is used by someone now, can't be freed                  |
|unused  |`d_inode` != NULL && `d_count` == 0 |the dentry is vaild and nobody use it, can be freed                |
|negative|`d_inode` == NULL                   |act as a cache to indicate the dentry is invalid now, can be freed |

### dentry cache

The dentry cache consists of three parts:
1. Lists of "used" dentries linked off their associated inode via the `i_dentry` field of the inode object.
2. A doubly linked "least recent used" list of unused and negative dentry objects.
3. A hash table and hashing function used to quickly resolve a given path into the associated dentry object.

The dcache also provides the front end to an inode cache, the icache. Inode objects that are associated with
dentry objects are not freed because the dentry maintains a positive usage count over the inode.

## *file* object

The file object is the in-memory representation of an open file. The object is created in response to the
`open()` system call and destroyed in response to the `close()` system call. The file object merely represents
a process's view of an open file.

# Data Structure Associated with Filesystems

* `file_system_type` describing the capabilities and behavior of each filesystem.
* `vfsmount` represents a specific instance of a filesystem, i.e., a mount point.

# Data Structure Associated with a Process

* `files_struct`, which is pointed by `files` in `task_struct`,
  contains all per-process information about open files and file descriptors.
* `fs_struct` contains filesystem information(root directory and pwd for example)
   related to a process and is pointed by the `fs` field in `task_struct`
* `mnt_namespace`, which is pointed by `mnt_namespace` field in `task_struct`, enable
  an unique view of the mounted filesystems on the system.
