# CyberThreat.Binary1
1. Find call to libc time in binary
```
        00011465 83 ec 0c        SUB        ESP,0xc
        00011468 6a 00           PUSH       0x0
        0001146a e8 a1 f0        CALL       time   
                 ff ff
        0001146f 83 c4 10        ADD        ESP,0x10
        00011472 83 ec 0c        SUB        ESP,0xc
        00011475 50              PUSH       EAX
        00011476 e8 d5 f0        CALL       srand  
                 ff ff
```
Output of time() goes to A register, and libc srand() takes seed arg from the top of the stack, so we can see that the seed is just the output of time(). (The argument put on the stack before time is where to store the result, by passing 0x0(NULL in C), we dont store it in memory). You could also come to the same conclusion by running the program twice in a second
2. Look for calls to libc rand()
There is only one reference to rand(), which is at imagebase+0x788. This means that there is one function that picks random numbers to be used. It's likely that something could happen later on to change the number, but in this case it doesnt.
```
        00010788 e8 e3 fd        CALL       rand                                             
                 ff ff
        0001078d 89 c2           MOV        EDX,EAX
        0001078f 8b 45 08        MOV        EAX,dword ptr [EBP + 0x8]
        00010792 01 d0           ADD        EAX,EDX
        00010794 8b 55 0c        MOV        EDX,dword ptr [EBP + 0xc]
        00010797 8d 4a 01        LEA        ECX,[EDX + 0x1]
        0001079a 99              CDQ
        0001079b f7 f9           IDIV       ECX
        0001079d 89 55 80        MOV        dword ptr [EBP + -0x80],EDX
```
Looking at the code we can see a constant at [EBP+0x8] is added to the number. I found this number to be 1 through debugging, if you wanted to be sure you could find it further back in the code when it is pushed onto the stack. Same goes with [EBP+0xc], which was 101d. The program said it would generate a number between 1 and 100. This looks like a way of enforcing the range.
In pseudo code this would be..
```
rand_num = rand() + 1;
divisor = 101 
result = rand_num % divisor 
```
3. Emulate the same process
This is easy as we already know the position the sequence starts in, if this was on a website etc where the program isn't ran again every time, this wouldn't work so easy.
Equivalent code in C for the pseudo code earlier, except we do it 5 times as we are asked 5 times:
```
int seed = time(NULL);
srand(seed);
for(int i=0;i<5;i++) 
{
    printf("%d\n", (rand() + 1) % 101);
}
```
4. Try it with the binary
```
$ ./unlocker < <(./sploit)
We're going to play a game. I'll generate 5 numbers between 1 and 100, guess them all and I'll give you the password
I've chosen a number, what's your guess?


Good guess, I chose 64!
I've chosen a number, what's your guess?


Good guess, I chose 82!
I've chosen a number, what's your guess?


Good guess, I chose 92!
I've chosen a number, what's your guess?


Good guess, I chose 38!
I've chosen a number, what's your guess?


Good guess, I chose 100!
Heh.. You won, here's the zip password: 6215587638e9e76a7f61bf799e50e832ea531308
$ unzip -p -P 6215587638e9e76a7f61bf799e50e832ea531308 flag.zip
CT19{Bad_Random_Is_Bad}
```


