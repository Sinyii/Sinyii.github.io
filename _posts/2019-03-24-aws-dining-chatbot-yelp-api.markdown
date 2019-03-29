---
layout: post
title:  "AWS dining chatbot (Yelp api)"
date:   2019-03-24 17:45:36 -0400
thumbnail: "/image/chatbot/chat_bot_page2.png"
categories: projects
---


### [Try the dining chatbot](https://dining-chat-bot2.auth.us-east-1.amazoncognito.com/login?response_type=code&client_id=7a56fi10nj9q599klq7tjog785&redirect_uri=https://dkqtjjobwip8d.cloudfront.net/dining_chat.html) 
+ This domain is provided by **AWS cognito** and the redirect url is provided by **AWS cloudFront**.
+ You can use [10 minutes email](https://10minutemail.com/10MinuteMail/index.html) to sign up or just use the sample account below.
+ **Sample account**:
    - user name: sinyi
    - pwd: 12345678
+ Check the code on my [github](https://github.com/Sinyii/AWS-dining-chatbot-w-yelp-api).

+ Further explainations:
    - Using AWS cloudFront is for preventing people to connect with my S3 bucket directly, otherwise they could touch my other stuff in this bucket.

### 1. First, login to through cognito to get permission of connecting with chatbot.

![login page](/image/chatbot/login_page.png)

### 2. Say hi to chat bot!

![say hi to chat bot!](/image/chatbot/chat_bot_page1.png)

### 3. Specify your requirement.

![then](/image/chatbot/chat_bot_page2.png)

### 4. Keep going...

![then](/image/chatbot/chat_bot_page3.PNG)

### 5. Got some recommendations of dining! (data collected from yelp api)

![final result](/image/chatbot/chat_bot_page4.PNG)

#### AWS Services used:
- S3
- Lex
- Lambda Function
- Cognito
- Cloudfront
- Other: [Yelp Fusion API](https://www.yelp.com/developers/documentation/v3)

#### References:
1. https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-with-identity-providers.html
2. https://regularshow.fandom.com/wiki/Pops?file=Pops_character.png
3. https://darkcode1.blogspot.com/2018/12/amazing-transparent-login-form-just-by.html

Will upload more detail asap!