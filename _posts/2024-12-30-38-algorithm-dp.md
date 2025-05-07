---
layout: post
title: "알고리즘 - DP 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
hidden: true
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

1, 2, 3 합을 구성할 때 순서가 관계 없기 때문에 **오름 차순 정렬**을 한다.  
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


<br>

<details>
<summary>BOJ 거리</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/12026)

<br>

### 난이도 : ⭐⭐
이중 반복문을 돌아 dp[] 1차원 배열에 보도블럭을 밟았을 때 에너지 최솟값을 저장했다.


### 코드
```java
import java.util.*;
import java.io.*;

public class Main{

    public static void main(String[]args) throws Exception{
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());

        int N = Integer.parseInt(stz.nextToken());

        stz = new StringTokenizer(bf.readLine());
        String [] arr= new String[N];
        String roads = stz.nextToken();

        int idx =0;
        for(String road : roads.split("")){
            arr[idx] = road;
            idx++;
        }

        int[] dp = new int[N];

        for(int i=0;i<N-1;i++){
            if(i!=0 && dp[i] ==0) continue;

            for(int j=i+1;j<N;j++){
                if(arr[i].equals("B") && arr[j].equals("O") || arr[i].equals("O") && arr[j].equals("J") || arr[i].equals("J") && arr[j].equals("B")){
                    if(dp[j] == 0) dp[j] = dp[i]+((j-i)*(j-i));
                    dp[j] = Math.min(dp[j],dp[i]+((j-i)*(j-i)));
                }
            }
        }
        int answer = -1;
        if(dp[N-1]!=0) answer = dp[N-1];
        System.out.println(answer);
    }
}
```
</div>
</details>

<br>

<details>
<summary><strong>평범한 배낭</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/12865)

<br>

### 난이도 : ⭐⭐⭐
dp[i][j] 이차원 배열을 선언했고 i번째 물건까지 고려하고 j무게를 최대로 했을 때 가치의 최대값을 저장했다.
예를들어 dp[3][2]는 3번째 물건까지 고려됐고 최대 무게가 2일 때 가치의 최대값을 나타낸다.
dp[2][2] 와 비교해 크거나 같기 때문에 `dp[i][j]=dp[i-1][j];` 로 초기화를 진행했다.

### 코드
```java
import java.io.*;
import java.util.*;


public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());
        int limit = Integer.parseInt(stz.nextToken());

        int[][]arr = new int[n+1][2];
        for (int i = 1; i<=n; i++) {
            stz = new StringTokenizer(br.readLine());
            int w = Integer.parseInt(stz.nextToken());
            int v = Integer.parseInt(stz.nextToken());

            arr[i][0] = w;
            arr[i][1] = v;
        }

        int[][]dp = new int[n+1][limit+1];

        for(int i=1;i<=n;i++){
           for(int j=1;j<=limit;j++){
               dp[i][j]=dp[i-1][j];
               if(j-arr[i][0]>=0){
                  dp[i][j]=Math.max(dp[i][j],dp[i-1][j-arr[i][0]]+arr[i][1]);
               }
           }
        }

        System.out.println(dp[n][limit]);
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>크리 보드</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/11058)

<br>

### 난이도 : ⭐⭐
4가지 선택지가 있기 때문에 모두 탐색하게되면 최대 4<sup>100</sup> 연산을 수행해야한다.
dp[i] 1차원 배열에 i번 눌렀을 때 출력되는 A의 최대값을 저장한다.  

1번 버튼이 아닌 2, 3, 4버튼으로 A를 출력하기 위해서는
전체 선택 -> 복사 -> 붙여넣기가 모두 수행되어야하기 때문에 최소 3번의 키보드가 더 눌려야한다.
3번 이후부터는 붙여넣기를 통해 출력이 가능하므로

`dp[i+j] = Math.max(dp[i+j] ,dp[i]*(j-1))`이 된다.  

1번 버튼을 통해 화면에 A하나만 출력할 수 있기 때문에 `dp[i+1]= Math.max(dp[i+1],dp[i]+1);`를 추가했다.

### 코드
```java
import java.io.*;
import java.util.*;


public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());

        long []dp =new long[n+1];
        dp[1] =1;
        for (int i=1; i<n; i++){
            dp[i+1]= Math.max(dp[i+1],dp[i]+1);
            for(int j=3;i+j<=n;j++){
                dp[i+j]=Math.max(dp[i+j],dp[i]*(j-1));
            }
        }
        System.out.print(dp[n]);
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>소형 기관차</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/2616)

<br>

### 난이도 : ⭐⭐⭐
소형 기관차가 3대이기 때문에 최대 N<sup>3</sup> 연산을 수행해야한다.

dp[i][j] 2차원 배열을 생성해서 i번쨰 소형 기관차, j번째 객차까지 고려했을 때 최대 손님수를 저장했다.

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{

    public static void main(String[]args) throws Exception{
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());

        int n = Integer.parseInt(stz.nextToken());

        int[] arr = new int[n+1];
        int[] sum = new int[n+1];
        stz = new StringTokenizer(bf.readLine());
        for(int i=1;i<=n; i++){
            int num = Integer.parseInt(stz.nextToken());
            arr[i] = num;
            sum[i] = sum[i-1] + num;
        }

        stz = new StringTokenizer(bf.readLine());
        int limit = Integer.parseInt(stz.nextToken());
        int[][] dp = new int[4][n+1];

        for(int i=1;i<=3;i++){
            for(int j=i*limit; j<=n; j++){
               dp[i][j] = Math.max(dp[i][j-1],dp[i-1][j-limit]+sum[j]-sum[j-limit]);
            }
        }
        System.out.println(dp[3][n]);

    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>동전1</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/2293)

<br>

### 난이도 : ⭐⭐⭐
dp 배열을 선언하고 dp[i] i금액으로 만들 수 있는 경우의 수를 1번 동전부터 누적함

### 코드
```java
import java.util.*;
import java.io.*;

public class Main {
    static int n,k;
    public static void main(String[] args) throws IOException {
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());

        n = Integer.parseInt(stz.nextToken());
        k = Integer.parseInt(stz.nextToken());
        int []dp = new int[k+1];
        int [] arr = new int[n+1];

        for(int i=1;i<=n; i++){
            stz = new StringTokenizer(bf.readLine());
            arr[i] = Integer.parseInt(stz.nextToken());
        }

        dp[0] = 1;
        for(int i=1; i<=n; i++){
            int num = arr[i];
            for(int j=num; j<=k; j++){
                dp[j] = dp[j]+dp[j-num];
            }
        }
        System.out.println(dp[k]);
    }
}

```
</div>
</details>


<br>

<details>
<summary><strong>진우의 달 여행 (Large)</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/17485)

<br>

### 난이도 : ⭐⭐⭐
dp[i][j][k] 3차원 배열을 생성 k의 방향으로 i,j의 접근했을 때 최소값을 저장

### 코드
```java
import java.io.*;
import java.util.*;

public class Main {
    static int[] tx = {1,1,1};
    static int[] ty = {0,1,-1};
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int n = Integer.parseInt(stz.nextToken());
        int m = Integer.parseInt(stz.nextToken());
        int [][] arr = new int[n][m];
        for(int i=0; i<n; i++){
            stz = new StringTokenizer(br.readLine());
            for(int j=0;j<m; j++){
                arr[i][j] = Integer.parseInt(stz.nextToken());
            }
        }
        int [][][]dp = new int[n][m][3];

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                Arrays.fill(dp[i][j], Integer.MAX_VALUE);
            }
        }

        for (int j = 0; j < m; j++) {
            for (int d = 0; d < 3; d++) {
                dp[0][j][d] = arr[0][j];
            }
        }
        for(int i=0; i<n; i++){
            for(int j=0;j<m; j++){
                for(int k=0; k<3; k++){
                    for(int l =0; l<3; l++){
                        if(k == l) continue;
                        int rx = i+tx[l];
                        int ry = j+ty[l];
                        if(ry<0 || rx <0 || rx>=n ||ry>=m ) continue;
                        if(dp[i][j][k] != Integer.MAX_VALUE) dp[rx][ry][l] = Math.min(dp[rx][ry][l],dp[i][j][k] + arr[rx][ry]);
                    }
                }
            }
        }
        int answer = Integer.MAX_VALUE;
        for(int j=0;j<m; j++){
            answer = Math.min(answer,Math.min(Math.min(dp[n-1][j][0],dp[n-1][j][1]),dp[n-1][j][2]));
        }
        System.out.println(answer);
    }
}
```
</div>
</details>

<br>

<details>
<summary><strong>동전</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/9084)

<br  
### 난이도 : ⭐⭐⭐
dp[i] 일차원 배열을 사용해 i원일 때 방법 수를 저장  
dp[i] = dp[i] + dp[i-coin]; 이 점화식이며 dp[0] =1을 입력해 반복문 수행

### 코드
```java
import java.io.*;
import java.util.*;

public class Main {
    static int[] tx = {1,1,1};
    static int[] ty = {0,1,-1};
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int n = Integer.parseInt(stz.nextToken());
        int m = Integer.parseInt(stz.nextToken());
        int [][] arr = new int[n][m];
        for(int i=0; i<n; i++){
            stz = new StringTokenizer(br.readLine());
            for(int j=0;j<m; j++){
                arr[i][j] = Integer.parseInt(stz.nextToken());
            }
        }
        int [][][]dp = new int[n][m][3];

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                Arrays.fill(dp[i][j], Integer.MAX_VALUE);
            }
        }

        for (int j = 0; j < m; j++) {
            for (int d = 0; d < 3; d++) {
                dp[0][j][d] = arr[0][j];
            }
        }
        for(int i=0; i<n; i++){
            for(int j=0;j<m; j++){
                for(int k=0; k<3; k++){
                    for(int l =0; l<3; l++){
                        if(k == l) continue;
                        int rx = i+tx[l];
                        int ry = j+ty[l];
                        if(ry<0 || rx <0 || rx>=n ||ry>=m ) continue;
                        if(dp[i][j][k] != Integer.MAX_VALUE) dp[rx][ry][l] = Math.min(dp[rx][ry][l],dp[i][j][k] + arr[rx][ry]);
                    }
                }
            }
        }
        int answer = Integer.MAX_VALUE;
        for(int j=0;j<m; j++){
            answer = Math.min(answer,Math.min(Math.min(dp[n-1][j][0],dp[n-1][j][1]),dp[n-1][j][2]));
        }
        System.out.println(answer);
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>포도주 시식</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/2156)

<br>

### 난이도 : ⭐⭐
dp[i] 배열을 선언해 i번째 포도주까지 고려시 최대양을 저장함  
세개를 연속해서 먹을 수 없으므로 dp[i]은 다음과 같이 나타낼 수 있음
- dp[i-1] 
- dp[i-2] + arr[i]  
- dp[i - 3] + arr[i - 1] + arr[i]

dp[i-2] + arr[i]를 예로들면 dp[i-2]까지 어떻게든 최댓값를 구했다면 arr[i-1]을 건너뛰고 
arr[i]를 더한 것이 연속 세번을 넘지 않기 때문에 후보가 됨


### 코드
```java
import java.io.*;
import java.util.*;


public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());

        int[] arr = new int[n + 1];
        int[] dp = new int[n + 1];
        for (int i = 1; i <= n; i++) {
            stz = new StringTokenizer(br.readLine());
            arr[i] = Integer.parseInt(stz.nextToken());
        }

        for (int i = 1; i <= n; i++) {
            if(i>=3){
                dp[i] = Math.max(dp[i - 1], Math.max(dp[i - 2] + arr[i], dp[i - 3] + arr[i - 1] + arr[i]));
            } else {
                dp[i] = dp[i-1]+arr[i];
            }
        }
        System.out.println(dp[n]);
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>징검다리 건너기</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/21317)

<br>

### 난이도 : ⭐⭐
dp[i][j] dp 2차원 배열을 선언해서 i번째 징검다리에 도착했을 때 최소 에너지값 , j는 매우 큰 점프 사용 여부를 저장

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    public static void main(String []args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());

        ArrayList<Integer>[] arr = new ArrayList[n];
        for(int i=0; i<n-1; i++){
            stz = new StringTokenizer(br.readLine());

            ArrayList<Integer> list = new ArrayList<>();
            int e = Integer.parseInt(stz.nextToken());
            int be = Integer.parseInt(stz.nextToken());
            list.add(e);
            list.add(be);
            arr[i+1] = list;
        }
        stz = new StringTokenizer(br.readLine());
        int vbe = Integer.parseInt(stz.nextToken());

        int [][] dp =new int[n+1][2];
        for(int i=0; i<dp.length; i++){
            Arrays.fill(dp[i],Integer.MAX_VALUE);
        }

        dp[1][0] = 0;
        dp[1][1] = 0;
        for(int i=1; i<=n; i++){
            if(i-1 >=1) {
                dp[i][0] = Math.min(dp[i][0],dp[i-1][0] + arr[i-1].get(0));
                dp[i][1] = Math.min(dp[i][1],dp[i-1][1] + arr[i-1].get(0));
            }
            if(i-2 >=1) {
                dp[i][0] = Math.min(dp[i][0],dp[i-2][0] + arr[i-2].get(1));
                dp[i][1] = Math.min(dp[i][1],dp[i-2][1] + arr[i-2].get(1));
            }
            if(i-3 >=1) {
                dp[i][1] = Math.min(dp[i][1],dp[i-3][0] + vbe);
            }
        }
        System.out.println(Math.min(dp[n][0],dp[n][1]));
    }
}
```
</div>
</details>



<br>

<details>
<summary><strong>쉬운 계단 수</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/10844)

<br>

### 난이도 : ⭐⭐
완전 탐색시 최대 2<sup>100</sup>의 연산을 수행함  
dp[i][j]에 길이가 i이고 j로 끝나는 계단 수를 저장

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    public static void main(String []args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int n = Integer.parseInt(stz.nextToken());

        long [][]dp = new long[n+1][10];

        for(int i=1; i<=9; i++){
            dp[1][i] =1;
        }

        for(int i=2; i<=n; i++){
            for(int j=0; j<=9; j++){
                if(j-1 >= 0) dp[i][j] = (dp[i][j] + dp[i-1][j-1])%1000000000;
                if(j+1 <=9) dp[i][j] = (dp[i][j] + dp[i-1][j+1])%1000000000;
            }
        }
        long answer = 0;
        for(int i=0; i<=9; i++){
            answer = (answer + dp[n][i]) %1000000000;
        }
        System.out.println(answer);
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>스티커 *</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/10844)

<br>

### 난이도 : ⭐⭐
dp[i][j] 
- j열의 위 스티커를 선택했을 때 최대값 i =1
- j열의 아래 스티커를 선택했을 때 최대값 i = 0  

0 3 0 4   
0 1 2 0  
만약 4에 해당하는 최대값을 구한다고 하면 1,2을 기준으로 최대값을 구하면된다. 3은 포함하지 않아도 되는 이유는 2에 최대값에 포함이 되기 때문


### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    static StringBuilder sb = new StringBuilder();
    public static void main(String []args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int t = Integer.parseInt(stz.nextToken());
        for(int i=0; i<t; i++){
            stz = new StringTokenizer(br.readLine());
            int n = Integer.parseInt(stz.nextToken());

            int [][] arr = new int[2][n+1];
            for(int j=0; j<2; j++){
                stz = new StringTokenizer(br.readLine());
                for(int k=1; k<=n; k++){
                    arr[j][k] = Integer.parseInt(stz.nextToken());
                }
            }
            int [][] dp = new int[2][n+1];
            dp[0][1] = arr[0][1];
            dp[1][1] = arr[1][1];
            for(int j=2; j<=n; j++){
                dp[0][j] = Math.max(dp[1][j-1],dp[1][j-2])+arr[0][j];
                dp[1][j] = Math.max(dp[0][j-1],dp[0][j-2])+arr[1][j];
            }
            sb.append(Math.max(dp[0][n],dp[1][n])).append("\n");
        }
        System.out.print(sb.toString());
    }
}

```
</div>
</details>


<br>

