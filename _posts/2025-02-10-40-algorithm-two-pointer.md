---
layout: post
title: "알고리즘 - 투포인터 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
hidden : true
---

<details>
<summary><strong>부분합</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1806)

<br>
### 난이도 : ⭐⭐


### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());
        int k = Integer.parseInt(stz.nextToken());

        stz = new StringTokenizer(br.readLine());
        int [] arr = new int[n];
        for(int i=0; i<n; i++){
            arr[i] = Integer.parseInt(stz.nextToken());
        }

        int start = 0;
        int end = 0;
        int sum = arr[0];
        int min = Integer.MAX_VALUE;
        while(true) {
            if(sum < k) {
                end++;
                if(end == arr.length) break;
                sum += arr[end];
            } else {
                min = Math.min(min, end-start+1);
                sum -= arr[start];
                start++;
            }
        }
        System.out.println(min==Integer.MAX_VALUE ? 0 : min);
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>용액</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/2470)

<br>
### 난이도 : ⭐
배열 첫번째 인덱스 , 마지막 인덱스 투포인터를 이용해서 용액 합을 계산함

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());

        stz = new StringTokenizer(br.readLine());
        int [] arr = new int[n];
        for(int i=0; i<n; i++){
            arr[i] = Integer.parseInt(stz.nextToken());
        }

        int min = Integer.MAX_VALUE;
        int start = 0;
        int end = n-1;

        int saveStart =0;
        int saveEnd =0;
        while(start != end){
            int num = arr[start] + arr[end];
            if(min > Math.abs(num)){
                saveStart = start;
                saveEnd = end;
                min = Math.abs(num);
            }
            if(num > 0) {
                end--;
            } else if (num < 0){
                start++;
            } else break;
        }
        System.out.print(arr[saveStart] + " "+ arr[saveEnd]);
    }
}
```
</div>
</details>


<br>

<details>
<summary><strong>소수의 연속합</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/1644)

<br>
### 난이도 : ⭐⭐

`for (int j = 2; j * j <= i; j++)` 를 통해 소수를 구함

### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());
        int []arr = new int[4000000];

        int index = 0;
        loop:
        for(int i=2; i<=n; i++) {
            for (int j = 2; j * j <= i; j++) {
                if (i % j == 0) continue loop;
            }
            arr[index++] = i;
        }

        int start =0;
        int end = 0;
        int count =0;
        int sum=0;
        while(true){
            if(sum <= n){
                if(sum == n) count++;

                if(arr[end] == 0){
                    break;
                }
                sum+=arr[end++];
            } else {
                sum-=arr[start++];
            }
        }
        System.out.println(count);
    }
}
```
</div>
</details>

<br>

<details>
<summary><strong>가장 긴 짝수 연속한 부분 수열 (large)</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/22862)

<br>
### 난이도 : ⭐⭐

end 시점을 미리 올려서 처리
### 코드
```java
import java.util.*;
import java.io.*;

public class Main{
    public static void main(String [] args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(stz.nextToken());
        int []arr = new int[4000000];

        int index = 0;
        loop:
        for(int i=2; i<=n; i++) {
            for (int j = 2; j * j <= i; j++) {
                if (i % j == 0) continue loop;
            }
            arr[index++] = i;
        }

        int start =0;
        int end = 0;
        int count =0;
        int sum=0;
        while(true){
            if(sum <= n){
                if(sum == n) count++;

                if(arr[end] == 0){
                    break;
                }
                sum+=arr[end++];
            } else {
                sum-=arr[start++];
            }
        }
        System.out.println(count);
    }
}
```
</div>
</details>



<br>

<details>
<summary><strong>합이 0</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/3151)

<br>
### 난이도 : ⭐⭐

세명 조합을 구해야하기 때문에 완전 탐색시  N<sup>3</sup>로 시간 초과 발생  
정렬 후 투 포인터를 사용해서 첫번째 학생은 고정(N)  ,이후 2명의 학생을 투 포인터로 탐색(N) 총 시간복잡도를 N<sup>2</sup>으로 줄일 수 있음

### 코드
```java
import java.util.*;
import java.io.*;
class Main{

    static  int[] arr;
    static long answer = 0;
    public static void main(String[] args) throws IOException {

        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int n = Integer.parseInt(stz.nextToken());
        stz = new StringTokenizer(br.readLine());
        arr = new int[n];
        for(int i=0; i<n; i++){
            arr[i]= Integer.parseInt(stz.nextToken());
        }
        Arrays.sort(arr);

        for(int i=0; i<n; i++){
            int start = i+1;
            int end = n-1;
            while (start < end){
                int sum = arr[i] + arr[start] + arr[end];
                if(sum == 0) {
                    int endVal = arr[end];
                    int startVal = arr[start];
                    int startCount = 0;
                    int endCount = 0;

                    if(endVal == startVal){
                        answer+= ((end-start+1) * (end-start))/2;
                        break;
                    }

                    while(endVal == arr[end]){
                        end--;
                        endCount++;
                    }

                     while(startVal == arr[start]){
                        start++;
                        startCount++;
                    }
                    answer+= startCount * endCount;
                } else if(sum > 0){
                    end--;
                } else {
                    start++;
                }

            }
        }
        System.out.println(answer);
    }
}
```
</div>
</details>



<br>

<details>
<summary><strong>겹치는 건 싫어</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/20922)

<br>
### 난이도 : ⭐⭐

### 코드
```java
import java.io.*;
import java.util.*;

public class Main {
    static int answer = 0;
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        int n = Integer.parseInt(stz.nextToken());
        int k = Integer.parseInt(stz.nextToken());

        int check[] = new int[100001];
        int arr[] = new int[n];
        stz = new StringTokenizer(br.readLine());
        for(int i=0; i<n; i++){
            arr[i] = Integer.parseInt(stz.nextToken());
        }

        int start = 0;
        int end = 0;
        int exceed = 0;
        while(true){
            if(exceed != 0){
                if(arr[start] == exceed) exceed = 0;
                check[arr[start]]--;
                start++;
                continue;
            }
            if(check[arr[end]]+1 > k){
                exceed = arr[end];
                continue;
            }
            check[arr[end]]++;
            end++;
            answer = Math.max(answer,end-start);
            if(end == n) break;
        }
        System.out.println(answer);
    }
}
```
</div>
</details>


<br>

<details>
<summary>블로그</summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/21921)

<br>
### 난이도 : ⭐⭐

### 코드
```java
import java.io.*;
import java.util.*;

public class Main {
    static int maxCount = 0;
    static int max = 0;
    public static void main(String[] args) throws Exception{
        BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(bf.readLine());


        int day = Integer.parseInt(stz.nextToken());
        int limit = Integer.parseInt(stz.nextToken());

        stz = new StringTokenizer(bf.readLine());

        int []visitor = new int[day];
        for(int i=0;i<day;i++){
            visitor[i] = Integer.parseInt(stz.nextToken());
        }
        int start = 0;
        int end = 0;
        int sum = 0;
        while(true){
            if(end == day)break;

            int count = end - start;
            if(count < limit){
                sum+=visitor[end];
                end++;
                if(count+1 == limit){
                    if(sum > max){
                        max = sum;
                        maxCount=1;
                    } else if (sum== max){
                        maxCount++;
                    }
                }
                continue;
            }
            sum-=visitor[start];
            start++;

        }
        if(max == 0) {
            System.out.println("SAD");
            return;
        }

        System.out.println(max);
        System.out.println(maxCount);
    }
}
```
</div>
</details>
