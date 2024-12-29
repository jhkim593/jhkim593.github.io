---
layout: post
title: "알고리즘 - DP 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
---

<details>
<summary>퇴사2</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/15486)

<br>
### 난이도 : ⭐

dp[i]: i일에 퇴사할 때 얻을 수 있는 최대 금액값을 저장했다.

### 코드
```java
import java.util.*;
import java.io.*;
public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));

        StringTokenizer stz = new StringTokenizer(bf.readLine());
        int count = Integer.parseInt(stz.nextToken());
        int[] time = new int[count+1];
        int[] fee = new int[count+1];

        int[] dp = new int[count+2];
        for(int i=1; i<=count; i++){
            stz = new StringTokenizer(bf.readLine());
            time[i] = Integer.parseInt(stz.nextToken());
            fee[i] = Integer.parseInt(stz.nextToken());
        }

        for(int i=1; i<=count+1; i++){
            //이전 날 최대 금액이 더 높으면 사용
            dp[i] = Math.max(dp[i-1],dp[i]);

            if(i==count+1) break;
            if(i+time[i]<=count+1){
                dp[i+time[i]] = Math.max(dp[i+time[i]], dp[i]+fee[i]);
            }
        }
        System.out.println(dp[count+1]);

    }
}
```
</div>
</details>


<br>

<details>
<summary>점프</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1890)

<br>
### 난이도 : ⭐

dp 2차원 배열 선언 후 각 요소에 접근할 수 있는 경로 수를 저장했다.

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{

    public static void main(String[]args) throws Exception{
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());

        int N = Integer.parseInt(stz.nextToken());
        int[][] arr = new int[N][N];
        long[][] dp = new long[N][N];
        for(int i=0;i<N;i++){
            stz = new StringTokenizer(bf.readLine());
            for(int j=0; j<N;j++){
                arr[i][j] = Integer.parseInt(stz.nextToken());
            }
        }
        dp[0][0] = 1;
        for(int i=0; i<N;i++){
            for(int j=0; j<N; j++){
                int jump = arr[i][j];
                if(jump!=0 && i+jump < N){
                    dp[i+jump][j] = dp[i+jump][j] + dp[i][j];
                }
                if(jump!=0 && j+jump < N){
                    dp[i][j+jump] = dp[i][j+jump] + dp[i][j];
                }
            }
        }
        System.out.println(dp[N-1][N-1]);
    }
}
```
</div>
</details>

<br>

<details>
<summary>1, 2, 3 더하기 4</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/15989)

<br>
### 난이도 : ⭐⭐

1, 2, 3 합을 구성할 때 순서가 관계 없기 때문에 오름 차순 정렬을 한다.  
오름 차순 정렬 했을 때 dp[i][j] 는 j로 끝날 때 i가 만들어지는 경우의 수이다.  
예를 들어 dp[4][2]는 2로 끝날 때 4가되는 경우의 수이다.  

... + 2 와 같은데 ...의 합은 2여야하며 오름 차순 정렬했기 있기 때문에 ...의 끝은 1 또는 2로 끝나야한다.  
식으로 나타내면 dp[4][2] = dp[2][1] + dp[2][2]가 된다.


### 코드
```java
import java.util.*;
import java.io.*;

public class Main{

    public static void main(String[]args) throws Exception{
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());

        int N = Integer.parseInt(stz.nextToken());


        for(int k=0;k<N;k++){
            stz = new StringTokenizer(bf.readLine());

            int n =Integer.parseInt(stz.nextToken());
            //dp 초기값 설정시 예외 발생 방지 위해 n+3
            int [][]dp = new int[n+3][4];

            dp[1][1] = 1;

            dp[2][1] = 1;
            dp[2][2] = 1;

            dp[3][1] = 1;
            dp[3][2] = 1;
            dp[3][3] = 1;
            for(int i=4; i<=n;i++){
                dp[i][1] = dp[i-1][1];
                dp[i][2] = dp[i-2][1] +dp[i-2][2];
                dp[i][3] = dp[i-3][1] +dp[i-3][2] + dp[i-3][3];       
            }
            System.out.println(dp[n][1]+dp[n][2]+dp[n][3]);
        }        
    }
}
```
</div>
</details>

<br>

<details>
<summary><strong>기타리스트</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1495)

<br>
### 난이도 : ⭐⭐

각 곡이 볼륨이 줄일건지 늘릴건지 2가지 경우가 있고 최대 곡 수가 50이기 때문에 최대 2<sup>50</sup> 연산이 수행된다. dp 2차원 배열의 값을 저장해 연산 수행을 줄였다.  
dp[i][j]에 i번째 곡이 연주될 때 볼륨 j가되면 1이 저장 되도록했다.

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{

    public static void main(String[]args) throws Exception{
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());

        int n = Integer.parseInt(stz.nextToken());
        int start = Integer.parseInt(stz.nextToken());
        int limit = Integer.parseInt(stz.nextToken());

        int[] arr = new int[n];
        stz = new StringTokenizer(bf.readLine());
        for(int i=0;i<n; i++){
            arr[i] = Integer.parseInt(stz.nextToken());
        }

        int[][]dp = new int[n+1][limit+1];
        dp[0][start]=1;

        for(int i=0;i<n; i++){
            for(int j=0;j<=limit;j++){
                if(dp[i][j]==0) continue;

                if(j+arr[i]<=limit){
                    dp[i+1][j+arr[i]] =1;
                }
                if(j-arr[i]>=0){
                    dp[i+1][j-arr[i]] =1;
                }
            }
        }
       int answer = -1;
        for(int i =0; i<=limit;i++){
            if(dp[n][i]==0) continue;
            if(i> answer) answer = i;
        }
        System.out.print(answer);
    }
}
```
</div>
</details>