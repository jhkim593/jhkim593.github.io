---
layout: post
title: "알고리즘 - DFS 문제 풀이 모음"
author: "jhkim593"
tags: Algorithm
---

<details>
<summary><strong>N-Queen</strong></summary>
<div markdown="1">

> [문제 링크](https://www.acmicpc.net/problem/9663)

<br>
### 난이도 : ⭐⭐⭐

### 코드

N*N 체스판에 퀸이 N개 들어가야하기 때문에 각 행마다 퀸은 반드시 들어간다.
index를 행 값을 열로 나타내는 1차원 배열을 생성했다.
arr[0] = 3일 때 (0+1)행 3열에 퀸이 존재함을 의미
```java
import java.util.*;
import java.io.*;

public class Main {
    static int[] arr;
    static int n;
    static int answer = 0;
    public static void main(String []args) throws Exception{
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer stz = new StringTokenizer(br.readLine());

        n = Integer.parseInt(stz.nextToken());

        arr = new int[n];
        dfs(0);
        System.out.println(answer);
    }
    public static void dfs(int cnt){
        if(cnt == n) {
            answer ++;
            return;
        }

        for(int i=0;i<n;i++){
            arr[cnt] = i;
            if(check(cnt)){
                dfs(cnt+1);                  
            }
        }
    }
    public static boolean check(int cnt){
        for(int i=0; i<cnt; i++){
            if(arr[i] == arr[cnt]) return false;
            if(Math.abs(cnt-i) == Math.abs(arr[cnt]-arr[i])) {
			return false;
		}
        }     
        return true;
    }

}
```
</div>
</details>
