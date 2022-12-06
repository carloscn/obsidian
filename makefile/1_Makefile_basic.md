# 1_Makefile_basic

There are the countless source files in a project, which are placed in serveral pathes by type, function and module. The makefile defines a series of rules to specify which files shall be built at first, which files shall be build later and which files shall be rebuilt.  The makefile likes a shell supporting complex functions. [^1][^2]

## grammer 

```Makefile
object:dependence
	<TAB> command line
```

* The `object` is the target to be compiled,  or an action.
* The `dependece` is a pre item performing the object depended.
	* One object allow to have multiple dependences.
* The `command` is the specific command under the performing object.
	* Each command occupy one line.
* By default, the frist object will be executed when no option is specified.  

Examples:
https://github.com/carloscn/clab/blob/master/macos/test_makefile/single/Makefile.hello

```Makefile
a:
	@echo "a"

b:
	@echo "b"

hello:
	@echo "hello world!"

clean:
	@rm -rf *.out

```

## common options

```bash
make [-f file] [options] [target]
```

By default, make command search `GUNmakefile` `makefiles` `Makefile` in current directory using as the make input file, except for that use option`-f` to specify a make file by filename.

* `-f`: specify a make file by file path
* `-v`: show the version number
* `-n`: perform the makefile, output commands record only, don't perform.
* `-s`: perform the makefile, don't show commands record.
* `-w`: show paths before and after execution.
* `-C`: show paths where the makefile is located.

## make flow

We need to set up a C build environment shown in the following picture, and you can find the original source code in https://github.com/carloscn/clab/tree/master/macos/test_makefile/single

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206210243.png)

The main.c file depends on add.c div.c mul.c sub.c, and the main.c is implement simple add or div functions by calling the sub-functions.

We written the makefile as follow:

```makefile
main :
	gcc add.c div.c sub.c mul.c main.c -o main.elf

clean :
	rm -rf *.o *.elf
```

It is a very simple makefile and it looks a bit silly.  When we changed a file only, all the objects would be re-compiled. If the project is very huge, it will take you much time on compiling.

So we need to optimize the makefile, make it easier and more convenient.

```makefile
# make -o to dump the VAL

OBJ = add.o div.o sub.o mul.o main.o
TARGET = main

$(TARGET).elf : $(OBJ)
	gcc $(OBJ) -o $(TARGET).elf

main.o : main.c
	gcc -c main.c -o main.o

add.o : add.c
	gcc -c add.c -o add.o

div.o : div.c
	gcc -c div.c -o div.o

sub.o : sub.c
	gcc -c sub.c -o sub.o

mul.o : mul.c
	gcc -c mul.c -o mul.o

clean :
	@rm -rf *.o *.elf
	@echo "make clean finish"
```

When one of these files changed, the optimized makefile would re-compiled the changed .c file. 

### using user env

Based on the simple makefile, we replaced `*.o` files to `OBJ` by user environment. The difference between the simple makefile and new makefile is shown in following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206211419.png)

### using sys env

* `$*`: the object name that is not includes extension name.
* `$+`: all the dependence files, seperated by space.
* `$<`: the first condition in rules.
* `$?`: the dependence files that the time later than object.
* `$^`: all the dependence files not repeated.
* `$%`: if the object is archive member, the variable indicate member name of the object.

Based on the user env makefile, we replaced `OBJ` and `TARGET` to `$^` and `^@` by system environment. 

```makefile
# make -p to dump the VAL

OBJ = add.o div.o sub.o mul.o main.o
TARGET = main

$(TARGET).elf : $(OBJ)
	gcc $(OBJ) -o $@

main.o : main.c
	gcc -c $^ -o $@

add.o : add.c
	gcc -c $^ -o $@

div.o : div.c
	gcc -c $^ -o $@

sub.o : sub.c
	gcc -c $^ -o $@

mul.o : mul.c
	gcc -c $^ -o $@

clean :
	@rm -rf $(OBJ) $(TARGET).elf
	@echo "make clean finish"
```

The difference between these makefiles is shown in following figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206212409.png)

### using build env

The build env can be shown by `make -p`.

* `AS`:  as
* `CC`:  gcc
* `CPP`: cc
* `CXX`: g++ or c++
* `RM`: rm -rf 

Based on the system env makefile, we replaced `gcc` to `$(CC)`by build environment. 

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206212535.png)

### phony object

```bash
.PHONY clean hello
```

The phony object is that whatever the file is changed or not, the object declared by phony must be performed.

If there is hello file on your makefile path, and hello is still a object  in your makefile.  When the hello is not declared by phony:

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206214114.png)

If the hello is declared by .PHONY, the object will be performed by make command whatever the file with the same name exists in the directory or not.

### mode matching

* `%.*:%.c`: .o files depends on .c files corresponding.
* `wildcard`: `$(wildcard ./*.c)` obtain the all .c file in the current directory.
* `patsubst`:`$(patsubst %.c, %.o, $(wildcard *.c))` replace .c to .o corresponding.

```makefile
.PHONY: clean hello

OBJ = add.o div.o sub.o mul.o main.o
TARGET = main

$(TARGET).elf : $(OBJ)
	$(CC) $(OBJ) -o $@

%.o : %.c
	$(CC) -c $^ -o $@

hello :
	@echo "hello test"

clean :
	@$(RM) $(OBJ) $(TARGET).elf
	@echo "make clean finish"
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206215055.png)

using the wildcard

```makefile
.PHONY: clean hello

OBJ = $(patsubst %.c, %.o, $(wildcard *.c))
TARGET = main

$(TARGET).elf : $(OBJ)
	$(CC) $(OBJ) -o $@

%.o : %.c
	$(CC) -c $^ -o $@

hello :
	@echo "hello test"

show:
	@echo $(wildcard *.c)
	@echo $(patsubst %.c, %.o, $(wildcard *.c))

clean :
	@$(RM) $(OBJ) $(TARGET).elf
	@echo "make clean finish"
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221206215142.png)


# Ref
[^1]: [ make.html ](https://www.gnu.org/software/make/manual/make.html#toc-Overview-of-make)
[^2]:[ makefile从入门到项目编译实战 ](https://www.bilibili.com/video/BV1Xt4y1h7rH?p=5&vd_source=edb82fc22cc42ae5edc71f0a1c41410e)













































