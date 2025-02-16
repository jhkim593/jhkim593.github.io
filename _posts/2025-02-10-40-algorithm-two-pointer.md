---
layout: post
title: "알고리즘 - 투포인터 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
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

> [문제 링크](https://www.acmicpc.net/problem/1806)

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
