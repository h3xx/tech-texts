# The 64 bit inode problem

*Author: Dr Michael Rutter (unknown; he wrote the [source
code](inode-problem/inode64.c), perhaps not the article)*

*Source: https://www.tcm.phy.cam.ac.uk/sw/inodes64.html*

Modern filesystems are starting to use 64 bit inodes, rather than 32 bit ones.
Recent versions of [XFS](http://www.xfs.org/) do so for filesystems of more
than 1TB, and other large filesystems such as [Lustre](http://lustre.org/) use
them too. NFS fully supports them too, so one can find them when mounting
remote filesystems.

Linux supports 64 bit inodes when applications are compiled 64 bit. For older
32 bit applications, they are supported, along with file sizes of more than
2GB, by non-standard calls such as `stat64()`, or by applications compiled with
options enabling large file support. As ten years have passed since the
introduction of Intel-compatible 64 bit processors (AMD's Opteron in 2003), and
the addition of large file support for 32 bit Linux, this should no longer be
an issue. However, there are still applications in use which are 32 bit and
have no large file support.

## The Issue

Two very common system calls, `stat()` and `readdir` return an inode number. If
the 32 bit version of these calls is used on a filesystem with 64 bit inodes,
then the call may fail with EOVERFLOW. However, most programs calling `stat()`
do not care about inode numbers, they are calling `stat()` for the other
information it returns, such as file type, length, ownership and permissions.
Similarly most programs calling `readdir()` simply wish to enumerate filenames
in a directory, and do not care about the directory's inode number.

I write "may fail" rather than "will fail" as the failure will happen only if
the particular inode number cannot be expressed in 32 bits. For example, on a
4TB XFS volume, if 64 bit inodes have been enabled, approximately one quarter
of the inode numbers will be expressible in 32 bits. So one can be lucky.

## Manifestations of Trouble

Various cryptic error messages result when programs encounter 64
bit indoes.

```
Error writing to foo: Value too large for defined data type
File I/O error: foo
An internal error occurred.
```

Of course, many of those errors can be caused by problems other than 64 bit
inodes. To see if 64 bit inodes are a likely cause one can check whether one
has them:

```
$ ls -li foo
12929830884 -rw-r--r-- 1 spqr1 roma 0 Aug 19 11:43 foo
```

The initial number is the inode number. If it is bigger than 4294967296 then it
is a 64 bit number rather than a 32 bit one. Ideally one also checks whether
the executable which is failing is indeed 32 bit. The check needs to be done on
the executable, not any shell script which launches it.

```
$ file /opt/acroread/Reader9/Reader/intellinux/bin/acroread
/opt/acroread/Reader9/Reader/intellinux/bin/acroread  ELF 32-bit LSB executable
```

## Solutions

### Recompile

Recompiling 64 bit should be safe. Recompiling 32 bit with
`_FILE_OFFSET_BITS=64` is slightly less safe, as the rest of the code needs to
respond correctly to the 64 bit values that it will now receive. (For instance,
`off_t` becomes a `long&nbsp;long`, and should not be truncated into a `long`
or an `int`).

Of course, if you don't have the source, this method won't
work.

### Ask the kernel to lie

Boot with the option `nfs.enable_ino64=0` (assuming the 64 bit inodes are
coming from an NFS mounted filesystem). This causes the kernel to return
non-unique 32 bit inode numbers to all applications, 32 bit or 64 bit, for all
NFS mounted filesystems. It can be a useful test that a problem is caused by 64
bit inode numbers, but is hard to recommend as some 64 bit applications might
be confused by the lie (which is to xor the top 32 bits of the inode number
with the bottom 32 bits).

### Use `LD_PRELOAD` to lie

One can use `LD_PRELOAD` to produce a wrapper which targets the three `stat()`
functions and `readdir()` that are the main cause of trouble, and returns a
mangled inode number as above. This has the advantage that it only impacts
certain 32 bit applications, and saner applications continue to get correct 64
bit inode numbers. It will also work with local filesystems, unlike the above
kernel parameter.

However, using `LD_PRELOAD` to intercept common system calls, and to lie, must
always come with a large health warning. Not doing this would be better!
However, for the brave who wish to experiment, there is some example code:

* [`inode64.c`](inode-problem/inode64.c)
* `inode64.so` (for the lazy)

It contains instuctions for compiling (either on a 32 bit platform, or on a 64
bit platform which supports 32 bit compilations). Then one simply needs to put
the resulting shared library somewhere, and then create an script called
something like `fix32` containing:

```
#!/bin/bash
export LD_PRELOAD=${LD_PRELOAD:+${LD_PRELOAD}:}/usr/local/lib/inode64.so
exec ${1+"$@"}
```

This can then be used in a form such as

```
$ fix32 acroread foo.pdf &
```

I repeat that this is given as an example for the brave, not with
any recommendation or guarantee that it will not eat your data.

(Note also that some versions of SuSE have an acroread script
containing the line

```
export LD_PRELOAD=$ACRO_INSTALL_DIR/intellinux/lib/suse-do-not-grab-server.so
```

This needs changing to

```
export LD_PRELOAD=${LD_PRELOAD:+${LD_PRELOAD}:}$ACRO_INSTALL_DIR/intellinux/lib/suse-do-not-grab-server.so
```

in order for this to work.)

## Further Reading

* [Does the world need 32 bit inodes?](http://blog.fmeh.org/2013/05/11/does-the-world-need-32-bit-inodes/)
* [XFS and inode size](http://xfs.org/docs/xfsdocs-xml-dev/XFS_User_Guide/tmp/en-US/html/ch06s06.html)
* [Adobe Reader and 64 bit inodes](https://forums.adobe.com/message/3521477)
