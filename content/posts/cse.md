01
Coping with complexity  
    Modularity 
    Abstraction 
    Layering 
    Hierarchy 
        Start with a small group of modules 
        Assemble them into a stable, self-contained subsystem with well defined interface Assemble a group of subsystems to a larger subsystem 

02
![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200427112323.png)

L1: Block Layer 
    Mapping: block number -> block data
    Super Block
        the size of block
        which block is free
        Kernel reads superblock when mount the FS 
L2: File Layer 
    inode (index node)
    A container for metadata about the file
        block, indrect block, double indirect block, triple indirect block

L3: inode Number Layer 
    Mapping: inode number -> inode

    ![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200427113436.png)
    inode number is enough to operate a file 

L4: File Name Layer 
L5: Path Name Layer 
    link
    unlink
    rename
L6: Absolute Path Name Layer 
L7: Symbolic LinkLayer 
    mount
        Record the device and the root inode number of the file system in memory 
        Record in the in-memory version of the inode for "/dev/fd1" its parent’s inode
    link

FAT (File Allocation Table) File System
    File is collection of disk blocks 
    In FAT: file attributes are kept in directory

03
Directly Dump a Directory
Two Types of Links
    Hard link
    Soft link
        Soft link can create cycle by SYMLINK("a", "a")
    ![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200427133254.png)
    --
    