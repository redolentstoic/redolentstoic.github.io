---
layout: post
title:  "Finding the cheapest book with AWS Lambda"
date:   2018-7-10
---

<style>
table{
    border-collapse: collapse;
    border-spacing: 0;
    border:2px solid #000000;
}

th{
    border:2px solid #000000;
}

td{
    border:1px solid #000000;
}
</style>

(Disclaimer: )

[BookDepository](https://www.bookdepository.com/) is well-known online book seller that belongs to Amazon.
BookDepository offers **free delivery worldwide**, so quite often I can acquire books
through BookDepository since I find better deals than in Amazon. 
Although I have been buying books with BookDepository for more than 7 years, I never really thought about what the **free delivery** option meant until recently. Recently, I was planning to buy the book [Designing Data-Intensive Applications : The Big Ideas Behind Reliable, Scalable, and Maintainable Systems](https://www.bookdepository.com/Designing-Data-Intensive-Applications-Martin-Kleppmann/9781449373320) (it's an awesome book by the way) and I thought of having a look if the price of this book is different based on the IP that I use to access BookDepository. So I used a VPN to login to BookDepository from Germany, as well as USA. Naturally the prices were different as you can see in the screenshots below. Buying the book from the USA the book costs $28.47, while from Germany (by converting euros to dollars) the book costs $36.31. So, I just went ahead and ordered the book using an American IP but by still delivering the book to a European address, thus saving $7.84. Whether or not this was a moral thing to do deserves is debatable and you can decide on your own. 

![Book from an German IP.]({{ "/images/germany.png" | absolute_url }})

![Book from an American IP.]({{ "/images/usa.png" | absolute_url }})

In other words, BookDepository offers *free delivery by incorporating the cost of the delivery in the price of the book*. In hindsight it seems this has been [widely known](https://www.quora.com/How-does-The-Book-Depository-offer-such-low-prices-and-remain-a-viable-business). But I became curious and I decided to answer the following two questions: 
* From which country can we most likely get the cheapest book?
* Are there books where the difference in price between different countries is extremely high (e.g., > $1000)?

To answer the aforementioned questions I decided I would go through 100,000 books and check the price of each book in different countries. This raises two subsequent questions: (i) How can we get different prices of a book depending on the country? and (ii) How can I get the URLs of 100,000 books. In what follows, I explained how I solved these issues. 

## Finding the prices using AWS Lambda
Assume we are given the URL of a book (e.g, https://www.bookdepository.com/Designing-Data-Intensive-Applications-Martin-Kleppmann/9781449373320) and we want to get a list of prices this book has in different countries.
One way would be to fire up a VPN, try different IPs, and write down the price of the book in each country. Naturally, such an approach would not scale for 100,000 books. Somebody could argue, that if I had the 100,000 book URLs I could just fire up a VPN, connect (let's say) from an American IP, get the prices of all the books from this American IP in an automated way, then connect from a German IP, etc. Such an approach could be automated and it would work fine. In any case, VPN services usually are costly (i.e., a few bucks a month), therefore I didn't want to follow such an approach. Additionally, I wanted to create an application that could give me the country where I can get the cheapest price for a book. Maybe I would like to use such an application in 3 months from now and I was not really in the mood of having a yearly VPN subscription just for this.

Since, I was not going to use a VPN service, I thought I could utilize Amazon EC2 instances, since they are offered in multiple regions (e.g., Frankfurt, London, etc.). In such a scenario, I would launch an instance in each possible region. Each instance will execute an application that given the URL of a book, connects to BookDepository, gets the price and returns it. Since instances reside in different regions I would get the prices for different countries. 
Still, such an approach has the issue that I would either need to have the instances always up and running which would work fine If I want to find the prices of 100,000 books. But if I want to find the prices of a book 3 months from now I would either need to have all instances up and running for 3 months which would be expensive, or launch and terminate the instance to just find the prices of a single book which would be expensive as well. For example, let us say I launch 10 total t2.nano instances in 10 different regions, even if I use them for 1 minute before I terminate them I would still need to pay [$0.067](https://aws.amazon.com/ec2/pricing/on-demand/).

Since I just wanted to get the price of the book, my problem seemed to perfectly fit the serverless paradigm.
So, I decided to use AWS Lambda. The main idea behind AWS Lambda is that we create a function that we upload to Amazon and then we can simply execute 

## Scraping BookDepository

## Results





