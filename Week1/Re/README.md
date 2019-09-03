# ollydbg

用ollydbg查看字符串，拿到flag

# babymips

用蛇(ghidra)打开, 找到main函数, ascii decode一下得到flag

# Crackme_for_begginer

od右键查看字符串，找到程序逻辑所在位置，然后单步运行找到判断flag的关键跳转，右键使用汇编使用nop填充，拿到falg。当然也可以直接逆password是什么：43967777

![](https://img-cjm00n.oss-cn-shenzhen.aliyuncs.com/Snipaste_2019-09-02_23-59-34.png)

# easy_reverse

同样根据字符串找到关键的位置，但是这次就需要认真的分析程序的逻辑了，把汇编学会，花点时间就能搞定。然后这题用IDA会更简单，但是为了防止大家只用IDA而不用od调试，因此加了一个小陷阱，就是程序在开始运行时会修改待比较的密文，和存放输入的数组的第21位为0x34。这一点在IDA静态分析时没那么容易看出来，但是用OD调试分析的话，几乎不会对你造成影响
主要加密逻辑

```
int len = strlen(buf);

if ( len > 20) return 0;

for (int i = 0; i < len; i++) {
		buf[i] = buf[i] ^ buf[i + 1] ^ buf[i + 2];
	}
	
for (i = 0; i < 22; i++) {
		if (buf[i] != cmp[i]) {
			printf("wrong!");
			return 0;
		}
	}
```
