---
categories:
- ccdak
- kafka
date: "2022-04-26"
author: Hugo Alves
resources:
- name: "featured-image"
  src: "kafka.png"
subtitle: A compilation of the study materials I used to pass the CCDAK certification.
tags:
- kafka
- certification
title: How I prepared for the CCDAK certification
---

The objective of this post is to compile the study materials I used to pass the CCDAK certification exam. All of my study was self paced without instructors, so I'll list all the materials and courses I used to prepare for the exam.

Before starting preparing for the CCDAK exam I didn't had a lot of experience with Kafka. 
Most of the experience I had was with the basic consumer/producer APIs and KSQL. I never had used Kafka Streams for example.


## Topics

To prepare for the certification I started with the [Kafka core concepts](https://www.udemy.com/course/apache-kafka/) Udemy course. This course introduces Kafka from the basic concepts until the most advanced APIs.

In my opinion if you never touched Kafka before you should do the exercises proposed during the course, otherwise you can skip them. The task of installing a cluster locally can be very time consuming. By using [Confluent Cloud](https://confluent.cloud/login) you can avoid setting up the cluster. New users of Confluent Cloud have some free credits that are enough for this purpose.


### Kafka Streams

After getting used to the core concepts I took the [Kafka Streams](https://www.udemy.com/course/kafka-streams/) course. In my opinion this is logical step to take after getting familiar with Kafka consumer/producer APIs.

Since I have never used Kafka Streams I dedicated some time to do the exercises proposed during the course. I found the exercises very useful to assimilate all the theory behind KStreams.

### Kafka Connect and Confluent Schema Registry

In my opinion these are more lightweight topics. I had some experience with these so I just reviewed the theory using the following Udemy courses.
But I believe that even if you don't have experience with Connect and schema registry it should be easy to quickly get the basics.

- [Kafka Connect](https://www.udemy.com/course/kafka-connect/)
- [Confluent Schema Registry](https://www.udemy.com/course/confluent-schema-registry/)

### KSQL

To me this was the more difficult topic. I spent most of my study time here.
Once again I recommend following [Udemy course](https://www.udemy.com/course/kafka-ksql/)


### Kafka Internals - advanced

If you want to understand better how kafka works internally I recommend to see the Confluent [Kafka 101](https://developer.confluent.io/learn-kafka/) tutorials. More concretely the [Apache Kafka® Internal Architecture](https://developer.confluent.io/learn-kafka/architecture/get-started/).

Despite not being a requirement for the certification I found these tutorials quite interesting to understand better how kafka works internally.

### Notes about Udemy courses

All of the previous mentioned courses can be found on Udemy for around 15€ each. If they are not priced like that you can either wait for Udemy promo (Udemy is 99% of the time in promo), or you can find coupons codes on the Author's [blog](https://medium.com/@stephane.maarek/how-to-prepare-for-the-confluent-certified-developer-for-apache-kafka-ccdak-exam-ab081994da78)

### Extra material 

I found the previous courses enough to get prepared for the exam. Although there are some details that can appear on the exam that they are not covered on the previously mentioned courses.

To complement the missing parts I read the Kafka The Definitive Guide book. This book is free and provided by [Confluent](https://www.confluent.io/wp-content/uploads/confluent-kafka-definitive-guide-complete.pdf).


## Practice

Before taking the exam I recommend you to test your knowledge with some practice tests. 

[This Udemy course](https://www.udemy.com/course/confluent-certified-developer-for-apache-kafka/) contains 3 tests with 50 questions each. I found the type of questions very similar to the questions I had on the real exam. If you are comfortable with these 3 tests easy you should have no issue in the real exam.

Then I also practiced using the questions of this [Cloud guru course](https://acloudguru.com/course/apache-kafka-deep-dive). To access these questions you must have a cloudGuru subscription, if you don't have one I believe it is not worth to pay it for just the practice tests.


## The Exam

I scheduled the exam on the confluent examinity platform. The exam costs 150$ (without VAT) and if you are in a hurry you can schedule it with 24h of advance, but I believe this has an extra cost.

On the day of the exam you will connect to the examinity platform and one proctor will connect to you on Zoom (You should install zoom before). During the exam the webcam, microphone and screenshare must be enabled.
The exam consists on 60 questions that have to be answered in 90 minutes.

After finishing the exam you will have to fill a small survey. After that you will get the result immediately. Confluent doesn't publish the minimum passing score to get approved, at the end of the exam you will simply get the pass/no pass information.

Good luck

{{< image src="https://api.accredible.com/v1/frontend/credential_website_embed_image/certificate/50217909" alt="ccdak result hugo alves" >}}