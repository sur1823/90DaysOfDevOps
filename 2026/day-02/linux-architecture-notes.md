Hello, Day-2

This .md explain the architecture of Linux in short but precise manner.

# Linux Architecture :
> user space : user apps like vim, docker, custom scripts, etc.

> shell : bash, zsh, etc

> system libs : wraps system-calls into c functions.

---------------------------------------

> ------- interface between user-space and kernel --------

---------------------------------------

> kernel

> Hardware.
> 

---------------------------------------

<img width="1912" height="2648" alt="linux architecture" src="https://github.com/user-attachments/assets/c086215b-1f7a-4977-9db1-58e845c5a250" />

---------------------------------------

# How processes are created and managed : 

Process Creation in Linux uses 2 syscalls. : fork() + exec()

Process Creation Flow Normally : 

When you type `ls` in bash, below is the workflow diagram for a Bash shell executing the #ls command:
```
+--------+
| pid=7  |
| ppid=4 |
| bash   |
+--------+
    |
    | calls fork()
    V
+--------+             +--------+
| pid=7  |    forks    | pid=22 |
| ppid=4 | ----------> | ppid=7 |
| bash   |             | bash   |
+--------+             +--------+
    |                      |
    | calls wait()         | calls exec() to run ls = exec("/usr/bin/ls")
    |   (blocks)           V
    |                  +--------+
    |                  | pid=22 |
    |                  | ppid=7 |
    |                  | ls     |
    |                  +--------+
    |                      |
    |                      | exits (status 0)
    |                      V
    +<---------------------+
    |
    | wait() returns, reaps zombie
    V
+--------+
| pid=7  |
| ppid=4 |
| bash   |
+--------+
    |
    | continues (shows prompt)
    V   
```

1.The Bash shell (parent, PID 7) calls fork(), creating a child process (PID 22). 
2.The parent (Bash) immediately calls wait(), which blocks (pauses) its execution. 
3.The child (PID 22) calls exec() to replace itself with the ls program. 
4.The ls program runs and eventually calls exit(0) or otherwise.
5.The child becomes a zombie. (The kernel keeps the process entry (as a zombie) to preserve the child's exit status so the parent can retrieve it later via wait())
6.The parent's wait() call completes, reads the exit status, and cleans up the zombie entry(reaped).
7.The parent (Bash) resumes and displays the command prompt.




- [ ] Unchecked task
- [x] Checked task
- [ ] Another unchecked task 


