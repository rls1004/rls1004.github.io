---
layout: post
title: "17 year old linux kernel bug"
image: /img/linux.PNG
tags: [Linux, Kernel, CVE]
---
2년 전에 대학원 수업 과제로 제출했던 보고서 내용입니다. 그 당시 17년 동안 존재해왔던 Linux Kernel 취약점에 대한 분석입니다. 지금은 19살이겠네요.  
  
*(written at 2018.10.24)*

## 17년 동안 존재해왔던 Linux Kernel 취약점 분석
Linux Kernel에 17년 동안 존재해온 취약점 발견, 영향을 받는 버전에 대한 주의 필요. (<4.17)  

### 1. CVE-2018-6554 : Denial of Service
Linux Kernel 4.17 이전 버전의 `net/irda/af_irda.c` 와 `drivers/staging/irda/net/af_irda.c` 에 존재 하는 `irda_bind()` 함수에서 메모리 누수가 발생  
AF_IRDA 소켓을 반복적으로 바인딩함으로써 local denial of service 유발 가능  

**Af_irda.c 에 구현된 irda_bind() 함수**
```c
static int irda_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len) {
    struct sock *sk = sock->sk;
    struct irda_sock *self = irda_sk(sk);
    ...	/* 이미 바인딩 되었는지 확인하는 작업 없음 */
    [1]	self->ias_obj = irias_new_object(addr->sir_name, jiffies);
    if (self->ias_obj == NULL)	return -ENOMEM;
    [2]	err = irda_open_tsap(self, addr->sir_lsap_sel, addr>sir_name);
    …
    [3]	irias_insert_object(self->ias_obj);
```
- 소켓을 bind 할 때, 이미 바인딩 되었는지 확인하지 않기 때문에 동일한 소켓을 반복적으로 바인딩 하고, 이전에 바인딩한 객체의 참조를 잃어버리게 됨. 소켓이 반복적으로 바인딩 되며 메모리 자원을 소모함.

### 2. CVE-2018-6555 : LPE
`irda_setsockopt()` 함수는 조건에 따라 새로운 객체를 할당하거나 기존 객체를 재사용 함. 소켓을 관리하는 더블 링크드 리스트에 기존 객체가 다시 삽입될 경우 충돌 발생.

|새로운 객체 삽입 (정상적인 경우)|기존 객체 삽입 (비정상적인 경우)|
|:-:|:-:|
|![insert](/img/202008/linux_00.png)|![reinsert](/img/202008/linux_01.png)|

해당 취약점을 이용하여 임의 주소에 객체의 주소값을 쓸 수 있음. 이를 통해 로컬 권한 상승 가능.  

**- insert 과정**

```c
element->q_next = (*queue);
(*queue)->q_prev->q_next = element;
element->q_prev = (*queue)->q_prev;
(*queue)->q_prev = element;
(*queue) = element;
```

**- close 과정**

```c
element->q_prev->q_next = element->q_next;
element->q_next->q_prev = element->q_prev;
if ( (*queue) == element)
  (*queue) = element->q_next;
```

<br>
<table>
<tr>
  <th colspan='2' style='text-align: center'>취약점 악용 과정</th>
</tr>

<tr>
  <td>①	소켓 세 개 bind</td><td>② 두 번째 소켓을 다시 bind</td>
</tr>
<tr>
  <td width='45%'><img src='/img/202008/linux_02.png'/>
  </td>
  <td width='55%'><img src='/img/202008/linux_03.png' width='75%'/><br>
  
  </td>
</tr>
<tr></tr>
<tr>
  <td>
  - head : s3<br>
  - s1->prev = s2, s1->next = s3<br>
  - s2->prev = s3, s2->next = s1<br>
  - s3->prev = s1, s3->next = s2
  </td>
  <td>
  - head : <font color='SALMON'>s2</font><br>
  - s1->prev = s2, s1->next = <font color='SALMON'>s2</font><br>
  - s2->prev = <font color='SALMON'>s1</font>, s2->next = <font color='SALMON'>s3</font><br>
  - s3->prev = <font color='SALMON'>s2</font>, s3->next = s2
  </td>
</tr>

<tr>
  <td>③	두 번째 소켓을 close</td><td>④ 세 번째 소켓을 close (first UAF)</td>
</tr>
<tr>
  <td><img src='/img/202008/linux_04.png'/></td><td><img src='/img/202008/linux_05.png' width='85%'/></td>
</tr>
<tr></tr>
<tr>
  <td>
  - head : <font color='SALMON'>s3</font><br>
  - s1->prev = s2, s1->next = <font color='SALMON'>s3</font><br>
  - s2->prev = x , s2->next = x <br>
  - s3->prev = <font color='SALMON'>s1</font>, s3->next = s2
  </td>
  <td>
  - head : <font color='SALMON'>s2</font><br>
  - s1->prev = s2, s1->next = <font color='SALMON'>s2</font><br>
  - s2->prev = <font color='SALMON'>s1</font>, s2->next = x<br>
  - s3->prev = x , s3->next = x
  </td>
</tr>

<tr>
  <td>⑤	두 번째 소켓 자리에 다른 객체 할당</td><td>⑥ 네 번째 소켓을 bind (arbitrary memory write)</td>
</tr>
<tr>
  <td><img src='/img/202008/linux_06.png'/></td><td><img src='/img/202008/linux_07.png'/></td>
</tr>
<tr></tr>
<tr>
  <td>
  - head : s2<br>
  - s1->prev = s2, s1->next = s2<br>
  - s2->prev = <font color='SALMON'>???</font>, s2->next = <font color='SALMON'>???</font><br>
  - s3->prev = x , s3->next = x
  </td>
  <td>
  - <font color='SALMON'>s2->prev->next = s4</font>
  </td>
</tr>
</table>


### 3. Patch
- CVE-2018-6554  
소켓을 bind 할 때, 이미 bind 된 소켓인지 확인하는 과정이 추가 됨

- CVE-2018-6555  
irda_setsockopt() 함수가 기존의 소켓을 재사용하지 않고 무조건 새로운 객체를  할당하여 사용하도록 수정 됨
