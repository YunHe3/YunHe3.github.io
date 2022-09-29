---

title: Eratosthenes筛法MPI优化

date: 2022-09-28 16:54:00 +0800

categories: [编程]

tags: [并行计算] [算法]

pin: true

author: YunHe3



toc: true

comments: true

typora-root-url: ../../YunHe3.github.io

math: false

mermaid: true



image:

---



最近在写mit6s081的第一个lab，其中有一道题要求使用fork和pipe来实现**并行**版本的埃拉托斯特尼**素数筛法**，其中对于pipe的管理颇有意思，故对该程序作一些分析。

先po我的代码：

```c
#define INTBYTE 4

int prime(int *p); /* p stand for a pair of pipe */

int
main()
{
	int p[2]; /* first pipe*/
    pipe(p);

    if (fork() == 0) {
        /* chlid process */
        close(p[1]);
        prime(p);
        close(p[0]);
        exit(0);
    } else {
        /* parent process */
        for (int i = 2; i <= 35; i++) {
            /* feeds 2 through 35 to child process */
            write(p[1], &i, INTBYTE);
        }

        close(p[1]);
				close(p[0]);
        wait((int *)0);
        exit(0);
    }
}

/* prime - read from p[0], write to newp[1] */
int 
prime(int *p)
{
	int transNum;
	int firstNum;
	int newp[2];

	if (read(p[0], &transNum, INTBYTE) == 0) {
		/* the last condition */
		exit(0);
	} else {
		firstNum = transNum;
		printf("prime %d\n", firstNum);

		/* creat a new pipe for the child process */
		pipe(newp);

		if (fork() == 0) {
      close(p[0]);
			close(newp[1]);
			prime(newp);
			close(newp[0]);
			exit(0);
		} else {
			/* parent process */
			while (read(p[0], &transNum, INTBYTE) != 0) {
				if (transNum % firstNum == 0) {
					/* drop the number */
					continue;
				} else {
					/* send the number to the child process */
					write(newp[1], &transNum, INTBYTE);
				}
			}
      close(newp[0]);
			close(newp[1]);
			wait((int *)0);
			exit(0);
		}
	}
}
```

这段代码理解起来很简单，每个进程内都有一个筛子，筛孔由读到的第一个数字决定，父进程将筛后的数字通过pipe传给它的直接子进程，之后子进程再重复父进程的工作，直到最后一个进程无法从其直接父进程中读取数字。

借用*Russ Cox*的图，这个过程可以抽象成如下过程：

![sieve](/assets/blog_res/2022-09-28-sievePrimeMPI.assets/sieve.gif)



重点谈谈实现该程序时对于pipe的管理，这同时是对fork和recursion的巩固。

该程序对于pipe的管理共有两套方案，分别在main函数中执行初始化与prime函数中参与迭代。main函数为代码入口，初始化数值并调用递归函数prime；prime函数的参数列表为一个int数组，表示一个pipe对，函数实现的功能为从指定pipe对读入待筛数据，创建新的pipe对并写入筛后的数据，同时以新的pipe对为参数调用prime。

1. main函数

   在函数的开始，创建了名为p的pipe，其中write end用于写入待筛的最初始值（为了简单，在代码中将其限制在35以内），read end用与第一次筛选。

   父进程部分

   * 代码块第20行```close(p[1])```，在执行完write操作后父进程关闭了pipe的write end，这里的关闭使得子进程中的read end能获知通信的结束（以pipe的read end为参数的read函数会在pipe的write end被关闭时返回0，否则会一直阻塞）。
   * 而由于fork会复制文件描述符，所以pipe的各个不被使用的end必须在父子进程中都得到释放，从而有第21行的```close(p[0])```

   子进程部分

   * 首先关闭不需要的p的write end```close(p[1])```，之后以p为参数调用prime
   * 由于```prime(p)```的调用结束意味着所有筛选过程的结束，当然也就意味着p这对pipe上的数据传输已经结束，因此可以关闭read end即```close(p[0])```

2. prime函数

   在函数运行到第48行后，创建了名为newp的pipe，这意味着此时该函数中有两个可使用的pipe，通过一系列关闭操作，在进入下一次递归时各pipe接口状态如下：

![image-20220928210748311](/assets/blog_res/2022-09-28-sievePrimeMPI.assets/pipe.jpg)

个人认为上图表达已足够清晰，故不做多余解释。

