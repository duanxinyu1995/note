1、输入包括两个正整数a， b（1<=a, b<=10^9), 输入数据包括多组

​	  输出a+b的结果

​	eg：![image-20210618093352866](C:\Users\段新宇\AppData\Roaming\Typora\typora-user-images\image-20210618093352866.png)

​	代码：



```c
#include <stdio.h>

int main(){

	int a, b;

	while(scanf("%d %d", &a, &b)!=EOF){

		printf("%d", a+b);

	}

}
```





2、输入第一行包括一个数据组数t(1<=t<=100), 接下来每行包括两个正整数a, b(1<=a, b<= 10^9)

​	   输出a+b的结果

​	eg.![image-20210618093809885](C:\Users\段新宇\AppData\Roaming\Typora\typora-user-images\image-20210618093809885.png)

C:

```c
#include <stdio.h>
int main(){
    int t , a, b;
    scanf("%d", &t);
    for(int i=0;i<t;i++){
        scanf("%d %d", &a, &b);
        printf("%d\n", a+b);
    }
    return 0;
}
```



3、 输入包括两个正整数a， b(1<=a, b<=10^9), 输入数据有多组， 如果输入为0 0 则结束输入

​		输出a+b的结果、

![image-20210618131524620](C:\Users\段新宇\AppData\Roaming\Typora\typora-user-images\image-20210618131524620.png)

C:

```c
#include <stdio.h>
int main(){
    int a, b;
    while(scanf("%d %d", &a, &b)!=EOF){
        if(a==0&&b==0)
            break;
        printf("%d", a+b);
    }
    return 0;
}
```



4、 输入数据包括多组 ， 每组数据一行， 每行的第一个整数为整数的个数n(1<=n<=100)， n为0 的时候结束输入，接下来n个正整数，即需要求和的每个整数

​		输出每组数据输出求和结果

![image-20210618141941930](C:\Users\段新宇\AppData\Roaming\Typora\typora-user-images\image-20210618141941930.png)

C：

```c
#include <stdio.h>
int main(){
    int n, num, sum=0;
    while(scanf("%d", &n)!=EOF){
        sum=0;
        if(n==0)
            break;
        for(int i=0;i<n;i++)
        {
            scanf("%d", &num);
            sum+=num;
        }
        printf("%d\n", sum);
    }
    return 0;
}
```



5、输入的第一行包括一个正整数t(1<=t<=100), 表示数据组数

接下来t行，每行一组数据。每行的第一个整数为整数的个数n(1<=n<=100)，接下来n个正整数，即需要求和的每个正整数。

​		输出每组数据求和的结果。

![image-20210618142350083](C:\Users\段新宇\AppData\Roaming\Typora\typora-user-images\image-20210618142350083.png)

C:

```c
#include <stdio.h>
int main(){
    int t, n, num, sum=0;
    scanf("%d", &t);
    for(int i=0;i<t;i++){
        scanf("%d", &n);
        sum=0;
        for(int i=0;i<n;i++){
            scanf("%d", &num);
            sum+=num;
        }
        printf("%d\n", sum);
    }
    return 0;
}
```

6、输入数据有多组，每行表示一组输入数据。每行的第一个整数为整数的个数n(1<=n<=100)，接下来n个正整数，即需要求和的每个正整数。

​		输出每组数据求和的结果。

![image-20210622211429518](C:\Users\段新宇\AppData\Roaming\Typora\typora-user-images\image-20210622211429518.png)

C:

```c
#include <stdio.h>
int main(){
    int n, num, sum=0;
    while(scanf("%d", &n)!=EOF){
        sum=0;
        for(int i=0;i<n;i++){
            scanf("%d", &num);
            sum+=num;
        }
        printf("%d\n",sum);
    }
    return 0;
}
```

7、输入数据有多组，每行表示一组输入数据。每行不定有n个整数，空格隔开。(1<=n<=100)

​	   输出每组数据的求和结果。

![image-20210622211642553](C:\Users\段新宇\AppData\Roaming\Typora\typora-user-images\image-20210622211642553.png)

C：

```c
#include <stdio.h>
#include <stdlib.h>

int main(){
    int num;
    long long sum=0;
    char s;
    while(scanf("%d" , &num)！=EOF){
        sum+=num;
        s=getchar();
        if(s=='\n'){
        	printf("%d\n", sum);
            sum=0;         
        }
    }
    return 0;
}
```

字符串排序1

​	输入两行，一行n，第二行为n个字符串，输出排序后的字符串，空格隔开，结尾无空格

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(){
    int n;
    char *nums=(char *)malloc(sizeof(char));
    char s;
    scanf("%d", &n);
    for(int i=0;i<n;i++){
        s=getchar();
        if(s==' ')
            continue;
        if(s=='\n')
            break;
        
    }
}
```

