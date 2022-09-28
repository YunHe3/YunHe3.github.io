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



最近在写mit6s081的第一个lab，其中有一道题要求使用fork和pipe来实现并行版本的埃拉托斯特尼素数筛法，其中对于pipe的管理颇有意思，故对该程序作一些分析

先po我的代码

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
		close(p[0]);
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
			close(newp[0]);
			while (read(p[0], &transNum, INTBYTE) != 0) {
				if (transNum % firstNum == 0) {
					/* drop the number */
					continue;
				} else {
					/* send the number to the child process */
					write(newp[1], &transNum, INTBYTE);
				}
			}
			close(p[0]);
			close(newp[1]);
			wait((int *)0);
			exit(0);
		}
	}
}
```

# 

