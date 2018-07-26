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
through BookDepository since I find better deals thatn in Amazon. 
Although I have been buying books with BookDepository for more than 7 years, I never really thought about what the **free delivery** option meant until recently. Recently, I was planning to buy the book [Designing Data-Intensive Applications : The Big Ideas Behind Reliable, Scalable, and Maintainable Systems](https://www.bookdepository.com/Designing-Data-Intensive-Applications-Martin-Kleppmann/9781449373320) (it's an awesome book by the way) and I thought of having a look if the price of this book is different based on the IP that I use to access BookDepository. So I used a VPN to login to BookDepository from Germany, as well as USA. Naturally the prices were different as you can see in the screenshots below. Buying the book from the USA the book costs $28.47, while from Germany (by converting euros to dollars) the book costs $36.31. So, I just went ahead and ordered the book using an American IP but by still delivering the book to a European address, thus saving $7.84.

![*Book from an German IP.*]({{ "/images/germany.png" | absolute_url }})

![*Book from an American IP.*]({{ "/images/usa.png" | absolute_url }})

In other words, BookDepository offers *free delivery by incorporating the cost of the delivery in the price of the book*. In hindsight it seems this has been [widely known](https://www.quora.com/How-does-The-Book-Depository-offer-such-low-prices-and-remain-a-viable-business). But I became curious and I decided to answer the following two quetions: 
* From which country can we most likely get the cheapest book?
* Are there books where the difference in price between different countries is extremely high (e.g., > $1000)?
In what follows, I explain 




