---
title: HTB - Diogenes Rage
author: akiju
date: 2022-10-13 13:00:00 +0700
categories: [Security, Tutorial]
tags: [CTF, Python, RaceCondition]
pin: true
---

# Hack The Box

## Category: Web

### Challange name: Diogenes Rage

Link: https://app.hackthebox.com/challenges/diogenes-rage

### 1. Intro

![Image](https://i.ibb.co/CVf9ncN/1.png)

- Users give a coupon to the selling machine and choose an item.
- A coupon is worth 1$

=> Question: Can I buy another high price item

### 2. Analysis

- Go to your browser's Web Developer tool / Network to check `requests.` Each cookie has a coupon.
![Image](https://i.ibb.co/gg0t99Q/2.png)
- Download and go to folder `web_diogenes_rage\challenge` to view the source code.
- View routing at `web_diogenes_rage\challenge\routes`, I focus on `api/purchase` and `api/coupons/apply` route. To get the challenge's flag, the user has to buy item C8
![Image](https://i.ibb.co/9y2YvYZ/4.png)
- In routing `api/coupons/apply`, the code creates a user object, each user has a username created by this code `tyler_${crypto.randomBytes(5).toString('hex')}` in web_diogenes_rage\challenge\middleware\AuthMiddleware.js and each user only has 1 coupon. The Program will query coupon names in database. If exist, the coupon is accepted. So, I try send more requests with data: {"coupon_name": "HTB_XXX"} (100-999) is not successful.
![Image](https://i.ibb.co/Z6t0B3H/3.png)
=> Question: How can I use only 1 coupon 1$ to buy the item that is worth 13$

- When request `api/coupons/apply` with data `{"coupon_name": "HTB_100"}`, the router use `db.getCouponValue` to get value of this coupon. Then use `db.addBalance` to add value of this coupon to the user. Then use `db.setCoupon` to add coupon code to the user.

=> Question: What happen if there are two requests with sharing same cookie at the same time at `db.addBalance` function ? This is Race condition.  

### 3. Exploit

**Step 1.** Use `/api/purchase` to generate new cookie.
```python
    sess = requests.Session()
    resp = sess.post(
            url=urljoin(host, "/api/purchase"),
            headers=headers,
            data=json.dumps({"item":"A3"})
    )    
```
**Step 2.** Exploit race-condition in `/api/coupons/apply` by multiple-threading code.
```python
    def exploit(cookies):
        resp = requests.post(
            url=urljoin(host, "/api/purchase"),
            headers=headers,
            cookies=cookies,
            data=json.dumps({"coupon_code":"HTB_100"})
        )
    
    with ThreadPoolExecutor(max_workers=300) as pool:
            for i in range(100):
                pool.submit(exploit, cookies)
```
I use class `ThreadPoolExecutor` from module `concurrent.futures`.
Thread Pool define:
```
A thread pool is a programming pattern for automatically managing a pool of worker threads.

The pool is responsible for a fixed number of threads.

It controls when the threads are created, such as just-in-time when they are needed.
It also controls what threads should do when they are not being used, such as making them wait without consuming computational resources.
Each thread in the pool is called a worker or a worker thread. Each worker is agnostic to the type of tasks that are executed, along with the user of the thread pool to execute a suite of similar (homogeneous) or dissimilar tasks (heterogeneous) in terms of the function called, function arguments, task duration, and more.
```
and `ThreadPoolExecutor` Python class is used to create and manage thread pools and is provided in the `concurrent.futures`

Lifecycle of the ThreadPoolExecutor

![Image](https://i.ibb.co/LQNhymH/10.png)

**Step 3.** By item C8 to get the `flag`.

### 4. Solution
