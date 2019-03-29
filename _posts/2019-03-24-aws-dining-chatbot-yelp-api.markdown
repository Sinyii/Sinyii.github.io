---
layout: post
title:  "AWS dining chatbot (Yelp api)"
date:   2019-03-24 17:45:36 -0400
thumbnail: "/image/chatbot/chat_bot_page2.png"
categories: projects
---


### =>[Try the dining chatbot](https://dining-chat-bot2.auth.us-east-1.amazoncognito.com/login?response_type=code&client_id=7a56fi10nj9q599klq7tjog785&redirect_uri=https://dkqtjjobwip8d.cloudfront.net/dining_chat.html) 

#### About the link:
+ There are **two urls** in the above link :
    - https://dining-chat-bot2.auth.us-east-1.amazoncognito.com
    - https://dkqtjjobwip8d.cloudfront.net/dining_chat.html
    - The first url is provided by **AWS cognito** for giving user autherization and the second url will redirect qualified user to the chatbot page. 
+ The redirect url **had been** the actual path of dining_chat.html in my S3 bucket, but I used **AWS cloudFront** to transform it into a safer format for preventing people to connect with my S3 bucket directly.

#### How to use:
+ You can use [10 minutes email](https://10minutemail.com/10MinuteMail/index.html) to sign up or just use the sample account below.
+ Sample account:
    - **user name**: sinyi
    - **pwd**: 12345678
    


### 1. First, login to through cognito to get permission of connecting with chatbot.

![login page](/image/chatbot/login_page.png)

### 2. Say hi to chat bot!
![say hi to chat bot!](/image/chatbot/chat_bot_page1.png)

### 3. Specify your requirements, you could say:
- *I would like to find a Japanese restaurant at New York at 5pm for 3 people*
- *I want to find a restaurant*
- *give me a restaurant*
- *I want food!*
- *restaurant*
- *or anything you like*
![then](/image/chatbot/chat_bot_page2.png)

### 4. Keep going...

![then](/image/chatbot/chat_bot_page3.PNG)

### 5. Get some recommendations of dining! (data is collected by yelp api)

![final result](/image/chatbot/chat_bot_page4.PNG)

#### What AWS Services was used:
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

#### Check out the code on my [github](https://github.com/Sinyii/AWS-dining-chatbot-w-yelp-api).

Will upload more detail asap!