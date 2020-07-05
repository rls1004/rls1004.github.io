---
layout: post
title: "Analyze Scam Email 3"
image: /img/scam.jpg
tags: [phishing, scam]
---

이번엔 피싱 사이트 유도가 아닌 스토리 텔링(?) 형식의 스캠 이메일을 다뤘습니다.  
스팸 메일과 스캠 메일을 구분하지 않고 사용했었는데 스팸 메일을 광고성 메일, 스캠 메일은 사기성 메일을 뜻하는거라 스팸 보다는 스캠이 맞을것 같아서 제목을 바꿨습니다 :)  

스토리 텔링식의 스캠 메일은 사이트 유도만 안 할뿐, 상대방을 속여서 돈이나 개인정보를 얻는 것을 목적으로 하는 것은 동일합니다.  

이 글에 등장하는 메일 전문은 제가 받은 이메일을 최대한 그대로 가져온 것입니다. 뒤죽박죽인 들여쓰기 정도는 보기 불편해서 수정했고, 해킹 당한 이메일로 보이는 것은 메일 주소를 일부 가렸습니다.  

분석이라기 보다는 재미로 모아봤습니다. 이메일에 등장하는 인물과 사연은 모두 허구입니다.  


# 서아프리가 은행 관리자의 메일

```
Title: Letter
Date: 2015-10-31 (토) 01:12
From: moha mmed<*******@naver.com>

Dear friend. 

I assume you and your family are in good health.

I am the Foreign Operations Manager at one of the leading generation bank here in Burkina Faso West Africa.

This being a wide world in which it can be difficult to make new acquaintances and because it is virtually impossible to know who is trustworthy and who can be believed, i have decided to repose confidence in you after much fasting and prayer.   It is only because of this that I have decided to confide in you and to share with you this confidential business.

In my bank; there resides an overdue and unclaimed sum of money worth some millions.  When the account holder suddenly passed on, he left no beneficiary who would be entitled to the receipt of this fund.  For this reason, I have found it expedient to transfer this fund to a trustworthy individual with capacity to act as foreign business partner.  Thus i humbly request your assistance to claim this fund.
Upon the transfer of this fund in your account, you will take fifty percent as your share from the total fund, while fifty percent will be for me.
Please if you are really sure you can handle this project, contact me immediately.

I am looking forward to read from you soon.

Thanks

Mohammed
```

서아프리카 은행 관리자인 모하메드라고 주장하는 분한테 온 메일입니다.  

> This being a wide world in which it can be difficult to make new acquaintances and because it is virtually impossible to know who is trustworthy and who can be believed

세상은 넓고 누가 신뢰할 수 있고 누가 믿을 수 있는지 아는 것은 사실상 불가능하다.. 명언입니다.  
모하메드씨는 단식과 기도 후에 저를 믿기로 어려운 결정을 하셨다고 합니다. 그러고는 비밀스러운 사업을 공유해주셨습니다.  

자신의 은행에, 수혜자를 남기지 않고 떠난 예금주가 있는데.. 음 무슨 원리인지는 모르겠지만 그 금액을 저에게 이체해서 반반 나누자는 내용입니다.  

더 대화 해 보지 않았지만 응답하게 되면 계좌번호를 포함한 개인정보를 요구하거나, 다른 이유를 들어서 이체를 요구할 수도 있을 것 같습니다.  

![국뽕](/img/202007/scam_00.jpg)  

모하메드씨가 네이버 메일을 사용한다는 것도 인상깊습니다.  

# 중국의 스웨터 공장

```
Title: golden river garment co.,ltd
Date: 2016-06-15 (수) 23:48
Frome: Panda<knitwear@googlemail.net>

Dear Managers,

We are sweater factory in Zhejiang Province, China.
Our company name: Zhejiang Golden River Garment Co.,LTD
300 workers, 30000 sqm workshops, 150000pcs monthly production, founded in 1994.
Certificates: SGS, ITS, BSCI, ISO etc.
Our top two customers: Next, Zara
We are producing both large orders and small orders.
If you have some sweaters need to be made, please contact us at any time.

Thanks.

Best regards,
Panda
Email:  niceknitwear@163.com
```

회사 PR을 하고 있습니다.  
회사 이름은 Zhejiang Golden River Garment Co. 이고 노동자는 몇 명, 장업장은 어떻고, 매달 얼마나 생산한다는 내용이 있습니다. 무슨 인증도 받았고, 고객사 자랑도 합니다. 스웨터가 필요한 경우에 문의 하라고 하네요.  

보낸 메일 주소와 문의 메일 주소가 서로 다른 도메인을 사용하고 있는 점이 특이합니다. google이라는 타이틀을 달고 의심을 피해보려는 시도 같기도 합니다. 하지만 구글이 제공하는 메일 도메인 중 googlemail<span></span>.com은 있지만 googlemail.net은 없습니다.  

마찬가지로 응답을 해보진 않았지만 스웨터를 판매하는 입장이니 스웨터에 대한 선불을 요구할 수 있을 것 같습니다.

```
Title: Golden River Knitwear
Date: 2016-07-11 (월) 14:40
From: Panda<panda@grknitwears.net>

Dear Sourcing / Buying manager / Designer,

We are sweater factory in Zhejiang Province, China.
300 workers, 30000 sqm workshops, 150000pcs monthly production, founded in 1994.
Have passed factory audits: SGS, ITS, BV, BSCI, MACYS, AURORA, SEARS, SEDEX, NEXT, ZARA, AVON, KOHLS, SMETA ETI etc...
Our top two customers: Next, Zara

We are producing both large orders and small orders.
If you have some sweaters need to do, please contact us at any time.
If you want to know more about our company, please email to:  jinheknit@163.com

Thanks.

Best regards,
Panda
Email:  niceknitwear@163.com
Email:  niceknitfashion@outlook.com
```

사업이 잘 안됐는지 한 달 쯤 후에 비슷한 내용의 메일이 또 왔습니다. 보낸 사람 이메일 주소도 바뀌었고 응답 이메일은 무려 3개를 써놨습니다. 물론 보낸 사람 이메일과 다릅니다.  

# 투자 상품 찾는 시리아 사람

```
Title: Hello,
Date: 2017-02-07 (화) 01:16
From: Hayyan Miran<hmsy111@outlook.com>

Dear Friend,

I have the intention of investing and purchasing of  properties with your
cooperation. My preferred choice of investment will be in residential
buildings but if you have better option of investment you consider
viable,you can recommend.My country Syria is currently in turmoil and
urgency is required for me to move my fund away to other nation.
Let me know your ideas

Thanks.

Hayyan Miran
```

```
Title: Hello!
Date: 2017-02-09 (목) 16:31
From: Hayyan Miran<hmsy111@outlook.com>

Dear Friend,

I have the intention of investing and purchasing of  properties with your
cooperation. My preferred choice of investment will be in residential
buildings but if you have better option of investment you consider
viable,you can recommend.My country Syria is currently in turmoil and
urgency is required for me to move my fund away to other nation.
Let me know your ideas

Thanks.

Hayyan Miran
```

3일 동안 같은 메일이 두 번 왔습니다.  
시리아가 혼란에 빠져서 자금을 다른 나라로 옮기고 싶은데 저에게 투자 상품과 아이디어를 묻는 내용입니다.  
안타깝게도 저는 리플이 떡상 하기 바로 전 날, 270원에 들어갔다가 300원에 좋다고 팔고 땅을 치고 후회하며 4000원에 다시 산 사람입니다.  

응답을 하게 되면 어떤 전개로 이어질지는 모르겠지만 응답 하는 것 만으로도 유효한 이메일이라는 것을 알려주게 됩니다.  

# 이혼모, 영국의 이슬람 은행원

### 첫 번째

```
Title: Hi dear,
Date: 2017-06-02 (금) 23:24
From: Maria Harriet Shaw<maria_isal@hotmail.com>

Hi dear,
How are you today? I am so glad that you accepted my friendship by responding back through your Email contact, I like to be open to you and also know more about you because i wish to build a rigid friendship and relationship with someone outside my Nationality and Race which i believe shall yield good fruit for both of us in the future depending on the effort we commit towards building the relationship, 

I am a woman that have seen life, i have been in the social circle for some years, It really does not matter one's age or color or achievement, what matters in our life is nothing but Care and expression and mostly expression of the heart.

This is the most important thing in life, to me the most beautiful thing created by God is never seen, it's only felt in the heart which is love. I have been hardworking all my life till now, i must think of something better to enjoy my life and probably have a family or maybe relocate and start investing in other things anyway. I will like to tell you little about myself,

My names are Maria Harriet Shaw. I'm a Britannia. I am a Single mother. Presently, i'm working as a Senior Audit/banker in Islamic Bank of Britain Plc United Kingdom now Al Rayan Bank.

I was married but my Ex Husband out of impatience got married to another woman which caused our divorce, but it's ok because i have to move on with my life since he accuses me of been so busy with my work in the bank. He says that i was not having time for him but had refused to understand that i was pursuing a goal, i tried to make him understand that soon i would resign and we shall have enough time for ourselves but he was impatient so i had to let him go, But it's over between us now, I am happy alone because I can be able to afford some of my needs by myself.

My daughter has been living with her Father since after our divorce. She's been the only family i got which is the reason i now joined Facebook to see if i can find someone honest and kind that we both can have plans on how to move on with our life's. I have a thought of change of life and relocating to your country to get into investment and maybe own a small company which i can be able to manage Enough for myself. How possible is that ?

I would like you to tell me more about yourself too.  I'll like to know you better, what you really do and your position in your work, your marital status and where you live now

I love to hear from you soon
Maria Harriet Shaw
```

이메일의 시작 부분에 답장해줘서 고맙다는 말이 있지만 처음 받는 메일 입니다. 자신의 사진이라며 어떤 여자 사진도 첨부했습니다.  
메일 내용에서는 제가 있는 나라로 이사와서 작은 회사를 소유하고 싶은데 어떻게 해야되냐고 묻네요. 제가 하는 일과 직장에서의 지위, 결혼 여부와 현재 살고 있는 곳도 물어봅니다. 제 사진을 포함한 개인 정보를 얻어낼 목적으로 보입니다.  

### 두 번째

```
Title: Best Regard
Date: 2017-09-02 (토) 06:01
From: Vilma Bormate Harriet<vilmabormate@hotmail.com>

Hi dear,
How are you today? I like to be open to you and also know more about you because i wish to build a rigid friendship and relationship with someone outside my nationality and Race which i believe shall yield good fruit for both of us in the future depending on the effort we commit towards building the relationship,
[...]
```

이름과 메일 도입부는 조금 다르지만 이후의 내용을 동일합니다. 첨부한 사진도 다른 여자의 모습이 담겨있습니다. 이분들이 아시면 놀라시겠죠..?ㅠ  

# 미군 여성

### 첫 번째

```
Title: Rv: HOW ARE YOU
Date: 2019-02-14 (목) 16:28
From: Mercedes Ramos Sixto<mramoss@agrorural.gob.pe>

My Dear ,
My name is (Lee Young) i want a serious relationchip, i am a pretty lady, kindly reply to my email below so that we cn get to know ourselves.
Thanks my dear lady
```

자신은 진지한 관계를 원하며 pretty lady라고 소개합니다. 메일 제목에 `Rv:`를 붙인 것은 답장 메일인 것처럼 보이기 위한걸까요?  
첨부파일로 사진이 있었는데 뭔가를 수여 받고 있는 동양인 미군 여성의 사진이었습니다. 파일명이 'DSAS' 였는데 군사 쪽에서 DSAS가 무엇을 의미하는지 아시는 분 있나요?  

### 두 번째

```
Title: It's nice meeting you
Date: 2019-03-24 (일) 20:41
From: Mariana Duran<duranmariana521@gmail.com>

我亲爱的，

感谢您的回应，感谢您的期待，我非常高兴看到您的邮件，感谢您和您周围的一切。我真的在寻求真正的爱情关系，我祈祷，如果你能以信任的方式向我表达爱意，我们将永远地生活在一起。我准备好成为你的，我相信内心的真爱，我需要生命中那个特别的人，对我有一颗慈爱的心。我相信年龄只是一个数字，但真正的爱来自于我对你心胸开阔的心。

关于我，我是Mariana Duran，美利坚合众国公民，但自从总统穆阿迈尔·卡扎菲逝世以来，他一直在利比亚执行维和任务，因为我的职业是服务于作为美国陆军的世界联合国我在肯塔基州浸信会儿童之家成长为孤儿，我今年30岁，单身而且从未结过婚，我附上了这封邮件的照片，我希望在回复邮件时你自己的照片。

更多关于我，我喜欢乒乓球，骑马，去看电影，慢跑和打高尔夫球作为最喜欢的运动，听爵士乐和摇滚音乐，非常浪漫和诚实，讨厌谎言，我认为我对人很敏感和善良，我的合作
- 工作人员总是说我善于保护人。我活着是自发的，我通常善于在任何我发现自己的位置上努力，我在工作中努力工作并且总是喜欢它，不管它给我这么多时间，我都很平静我的心，总是祈求上帝给我整个宇宙中最甜蜜的人，和我在一起很有意思，这意味着我是单身但是打算让我的一个人安顿下来，我独自一个人留在军营里美国。

在利比亚的军营，我们不允许使用手机，我们只使用无线电信息和电子邮件通信，所以请让我们继续通过电子邮件进行通信。我想更多地了解你，你的喜欢，不喜欢和生活经历，请你在回到我身边时给我发至少一张照片。

向那里的家人和朋友致以问候。

此致，

玛丽安娜杜兰
```

리비아 군대에 있는 30세 독신 미국 시민권자에게 중국어로 메일이 왔습니다.  
메일의 첫 부분에 답장해줘서 감사하다고 적혀있지만 처음 받는 메일입니다. 그 외에는 살아온 내용과 취미가 적혀있는데요, 탁구, 승마, 영화보기, 조깅, 골프, 재즈, 록 음악 듣기를 즐기고 자기는 낭만적이고 친절한 사람이며 거짓말을 증오한다고 합니다.  
제가 좋아하는 것과 싫어하는 것, 인생 경험에 대해 알고 싶다고 하네요.  

사진은 무려 3장을 첨부했습니다.  

### 세 번째

```
Title: Hello
Date: 2020-03-09 (월) 23:28
From: lisabrown<annbewmil@gmail.com>

I am lisa brown from united states of America.  
i have something very special i want to tell you
please get back to me on my email address
(lisabrownunitedstates@outlook.com)
so that i will tell you more details and my picture
Thanks.
```

특별하게 할 말이 있응니 답장달라며 이메일 주소를 알려줍니다. 보낸 사람 이메일과 다릅니다.  

어떤 전개로 사기를 치는지 궁금해서 이때 처음으로 응답을 해봤습니다.  

![응답](/img/202007/scam_01.PNG)

길게 말하긴 어려워서 핵심만 전달했습니다.  
미국 여성 사진이 첨부 된 답장도 왔습니다.  

```
Title: IT IS MY PLEASURE MEETING YOU
Date: 2020-03-12 (목) 00:20
From: lisa brown<lisabrownunitedstates@outlook.com>

My Dearest,

Thanks for your kind response and i am  happy after reading your mail.
It is my pleasure meeting you, I hope all is well with you and how are you enjoying your day? as i told you earlier in my previous letter on  my name is Lisa Brown, However, i really want to establish a true relationship with you and even to have a business partnership as well.

I am US military officer currently in Libya, and i will like to get you acquainted because I am a loving, honest and caring person with a good sense of humor, I enjoy meeting and sharing views with people and also knowing their way of life, I also enjoy watching the sea waves and the beauty of the mountains and everything that nature has to offer.

My dear,I want you to know the problems we are facing here in Libya as we are being attacked everyday by insurgents and car bomb explosion, but during our rescue mission we came across a safe box that contain huge amount of money that belongs to the supporters of the over thrown government of Libya, which I believe was money meant
for importation of weapons and ammunition, and it was agreed by my Army officers troop present to share the money among us and which we did.

My share is $4,560,000 (Four  Million Five Hundred and Sixty Thousand united states Dollars ) I am seeking your assurance and assistance to secure my share of the money out of this country ( Libya ) for safe keeping on my behalf.

If you can trusted, I will really want you to assure me that if this money gets to you in your country that you will not disappoint me, until when I will come to your country to collect the money from you. I would also want you to know that i have made a solid arrangements with a reliable security and delivery company and they have promised to corporate to deliver the funds through their diplomatic method to any of my choosing destination.

This delivery is going to be handle legally by the security company and there will not be any form of risk involve during the process and the money will be packaged safely  in a military truck box case. and I have decide to compensate you with 25% of the total money once you received the money, while the rest balance shall be for my investment capital in your country.

Meanwhile; I do not know how long we going to remain here and my fate since I have survived two bomb attack here, which prompted me to search out for a reliable and trust worthy person to help me receive and invest the Fund, because I will be coming over to your home country to invest and start a new life not as a soldier anymore.

Please do not discuss this matter to a third party, and if you are not interested to this business please kindly delete this letter from your email box to avoid any leakage of this information and it will be dangerous to me based on my position here.

I hope my explanation is very clear but should in case of any question, please feel free to ask via email only because we are not allow to make use of mobile phone, we only make use of radio message.
I wait for your immediate reply.
Love from.
lisa brown
```

리사 브라운은 리비아에 있는 미군 장교이고 폭탄 공격들 속에서 새로운 삶을 찾아 떠나고 싶어합니다. 어떻게 어떻게 해서 돈을 모아놨는데 군용 트럭 박스 케이스에 포장해서 보낼테니 자기가 찾으러 갈 때까지 맡아달라고 합니다.  
이 메일을 제 3자와 논의하지 말고 메일 상자에서도 삭제하라고 합니다. 뭘 요구하는 말들도 없고.. 이런 자극적인 말들 때문인지 뭔가 흥미진진해져서 주소를 알려주면 되냐고 답장을 또 보냈습니다.  

그에 대해 답장이 또 도착했습니다.  

```
Title: PLEASE SEND ME YOUR PERSONAL DETAIL
Date: 2020-03-12 (목) 22:06
From: lisa brown<lisabrownunitedstates@outlook.com>

My Dearest,

My beloved and how is your health, i hope you are healthy and fine?

I want to say thanks to you for making out time to write to me again, the content of your letter are well understood, i do not want you to see this business as some thing that will bring problem to you, but i want you to trust and believe me when i say that there is nothing to worry about in this deal, honestly i have made every necessary arrangement that will lead to safe delivery of the cash box to you without any form of problem nor risk, all the necessary arrangement that will lead to safe delivery of the cash box to you without any form of problem nor risk have already been made.

Am a woman that does things in accordance to the directives of my spirit, i chose you to be my partner and also help me in receiving this cash box because my spirit has been bearing me witness that you are the rightful person for me and i know you will not disappoint me, i chose to use you as partner in this deal because am not permitted to send package down to

any of my friends nor relatives since am still in the military camp outside USA, if i eventually send out any thing to the USA it will be suspicious and i will be query for the action, so i chose you because i know you are the ideal person for this deal as foreigner who is not a USA citizen.

Please put away fear or doubt and make up your mind to help me in this matter, i promise you you will not regret been part of this matter, i do not know what else i can say to convince you and make you to believe me, but i pray that God will give you the grace to make up your mind
One more time i want to let you know that there is no complication on this matter, if you follow my instruction every thing will go smooth and well in the process of receiving the cash box, Please try and keep this matter between you and i, because if you and i should agree on one thing with our heart with seriousness then we must surely achieve success at the end
Please i want to remind you once again that every arrangement towards this project is intact between both of us and on no account should you let the '' Security and delivery company '' to know the content of the box, remember that the consignment was registered as diplomatic package to the security company and that is what they believe to be in the box, so you should not let them know that the content of the box is money.

Please I will like you to send your contact information to me so that i can forward to the security company to enable them proceed to your country for the final delivery of the cash box to your door step

Send your below information to me urgently;

Your full names: ...........
Your address: ..............
Post code: .................
City: ......................
Your Age .................
Country: ...................
Telephone number: ................
Profession: ................
 
Once am through here with my official assignment i will come over to meet you one on one in your country and after that we shall decide on how to carry on with our life, but for now please i will appreciate us to be more focus on this issue of you receiving the cash box and safe keeping it on my behalf till i come over then i will handle the depositing of the money in the bank by myself.
I await your immediate response
Love and care from,
Lisa Brown
```

걱정 말라는 내용과 박스의 내용이 돈이라는 것을 알리지 말라는 내용이 있습니다. 보내야 될 정보는 전체 이름, 주소, 우편 번호, 나이, 핸드폰 번호, 직업 등 입니다.  

궁금증이 풀려서 여기까지만 했습니다.  


# 답장 유도

```
Title: FURTHER DETAILS OF MY PARTNERSHIP (OPEN ATTACHED MESSAGE
Date: 2019-05-31 (금) 05:14
From: Cheng-Chuan Kang<dir.chuan6@gmail.com>

Dear Friend,

Hope all is well with you and your Family, thank you for the response
to my email, sequel to my proposal to you, due to confidentiality I
have attached to this email a thorough explanation of this partnership
for your understand, do open the attached email, read diligently and
get back to me immediately.

I look forward to your urgent response, confirm your understanding and
your commitment to continue this partnership with me.

Regards,
Dir. Cheng-Chuan Kang
```

첨부 된 이메일에서 파트너십에 대한 내용을 읽고 답장 달라는 내용입니다. pdf가 첨부되어 있었는데 타이완 비지니스 은행에 대한 내용이었고 실제 해당 은행의 문서로 보였습니다. 메일에 나타난 이름도 실제 해당 은행의 디렉터였습니다.  
활성화 이메일을 파악하기 위한 메일일까요? 아니면 pdf에 뭔가 있는데 제가 분석을 못 한 걸까요?  
  

```
Title: Waiting for reply
Date: 2019-06-28 (금) 19:29
From: Jean edo<jamesobiaha01@gmail.com>

Dear Friend,
I am confirming if you received my previous mail regarding you having
the same last name with my late client, who leaves a large sum of money.
Please mail me for more
information:
Full Name:
Tel/Mobile No:
Regards,
Barr.Jean Edo, Esq
```

저와 같은 성을 가진 고객이 많은 돈을 맡겼는데 그게 저인지 아닌지 구분이 안가는 그런 상황을 말하고 있는 것 같습니다. 제 정보와 전체 이름, 핸드폰 번호를 요구합니다.  
  

```
Title: Hello,
Date: 2019-12-28 (토) 00:48
From: austinjames643@yahoo.com<austinjames643@yahoo.com>

Hello,

Please my name is Miss. Isabella Smith, a citizen of United States [USA] presently living in London [UK] as a Regional Branch Manager of Citibank UK.

I have a business to discuss with you, kindly get back to me for more details.

Reply to my private email: pw024696@gmail.com


Best Regards,

Miss Isabella Smith
Cell +447537180804
```

영국 시티 뱅크 매니저로, 영국에 거주하고 있는 미국 시민권자 스미스 입니다. 저와 의논할 사업이 있으니 개인 메일로 회신 달라고 합니다. 보낸 사람 메일과 회신용 메일이 다릅니다.  
  
---

모두 스캠 조심하세요 :)