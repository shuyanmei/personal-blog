---
title: "Create A Dynamic Order Book with Python"
description: "A short description"
date: '2018-08-26'
layout: 'posts'
featured: true
---

An order book is an electronic list of buy and sells orders for a specific security or financial instrument, organized by price level. The order book lists the number of shares being bid or offered at each price point. A dynamic order book is constantly updated in real time based on orders received from market participants such as banks, proprietary trading firms and retail investors.

Here I am trying to show how to create a dynamic order book with Python.

Dynamic Order Book - Example:

The input data follows the following format.

|Time Stamp   | Message Type   |   Order Reference |  Buy/Sell Indicator |  Shares| Stock  |  Price  |  Broker|
| :-------------: |:-------------:| :-------------:|:-------------:|:-------------: |:-------------:| :-------------:|:-------------:|
| milliseconds from midnight| Add Order |Order Reference  |'B' for Buy & 'S' for Sell  |Number of shares |Stock symbol |Price of the order| Broker number(3 digits) |



Message 1:
[35474335A 10000002S 500FGH 221000039]

Message 1 represents order was added at 09:51:14 AM (i.e. 35474335 milliseconds from midnight), the reference number is 10000002. The Buy/Sell indicator indicates that it is a sell order. The number of shares is 500 for stock FGH. The price of order is 22.10 and the broker number is 039.
 
| Best Bid Price  | Best Bid Size   | Best Ask Price  | Best Ask Size |
| :-------------------:      |     :-----------------------:| :---------------:|:----------:|
| |  | 22.10 | 500|


Message 2:
[35474336A 10000023S   100FGH           221500009]

| Best Bid Price  | Best Bid Size   | Best Ask Price  | Best Ask Size |
| :---------: |:-------------:| :-----:|:-----:|
| |  | 22.10 | 500|
| |  | 22.15 | 100|

Message 3:
[35475397A 10000024S   100FGH           221000080]

| Best Bid Price  | Best Bid Size   | Best Ask Price  | Best Ask Size |
| :---------: |:-------------:| :-----:|:-----:|
| |  | 22.10 | 600|
| |  | 22.15 | 100|


Message 4:
[35475398A 10000025S   100FGH           224800007]

| Best Bid Price  | Best Bid Size   | Best Ask Price  | Best Ask Size |
| :---------: |:-------------:| :-----:|:-----:|
| |  | 22.10 | 600|
| |  | 22.15 | 100|
| |  | 22.48 | 100|

Message 5:
[35475441A 10000027B   800FGH           220000079]

| Best Bid Price  | Best Bid Size   | Best Ask Price  | Best Ask Size |
| :---------: |:-------------:| :-----:|:-----:|
|22.00 |200  | 22.10 | 600|
| |  | 22.15 | 100|
| |  | 22.48 | 100|

Message 6:
[35475447A 10000028B   800FGH           218500079]

| Best Bid Price  | Best Bid Size   | Best Ask Price  | Best Ask Size |
| :---------: |:-------------:| :-----:|:-----:|
| 21.85| 800| 22.10 | 600|
|22.00 |200  | 22.15 | 100|
| |  | 22.48 | 100|

Message 7:
[35478449D 10000025   100]

| Best Bid Price  | Best Bid Size   | Best Ask Price  | Best Ask Size |
| :---------: |:-------------:| :-----:|:-----:|
| 21.85| 800| 22.10 | 600|
|22.00 |200  | 22.15 | 100|
| |  | | |

Suppose the above data is input and I created a function to build the dynamic order book.

The logic is very straightforward here. First, create dictionaries in python to store ask and bid price and size. Then extract the information such as price, share, order id and buy/sell indicator from the data and store these in a dictionary called order history. Since the data formats are different for the type of 'Cancel' and 'Add', consider the extraction in two cases.  

1) If the message type is 'Add', check if the price was listed before. If the price was listed before, then simply add the new size to the existing one. If not, create a new record to store this information. 

2) If the message type is 'Cancel', locate the 'Add' record based on order id and then subtract the corresponding amount of share.

Once all the information is collected, sort the ask table in ascending order and sort the bid table in descending order before concatenating them.

```python
def dynamicBookOrder(data):
    ask={}
    bid={}
    order_history={}
    for i in range(data.shape[0]):
        mess_type=data["message type"]
        ###if message type is not cancel, extract the price, share ,order id and buy/sell indicator
        if mess_type[i] != "D":
            price=float(int(data["Broker"][i][0:4])/100)
            share=int(data["share"][i][0:3])
            order_id=data["order_id"][i]
            bs_ind=data["BS indicator"][i]
            order_history[order_id] = { "message_type": mess_type,
                                           "bs_ind": bs_ind, "share": share, 
                                           "price": price}
            if bs_ind=="S":
            ###if the price is listed before, we simply added the share amount to it, otherwise, add a new row for the new price and amount
                if price in ask.keys():
                    before_share=ask[price]["share"]
                    ask[price]={"share":before_share+share,"price":price}
                else:
                    ask[price]={"share":share,"price":price}
            elif bs_ind=="B":
                if price in bid.keys():
                    before_share=bid[price]["share"]
                    bid[price]={"share":before_share+share,"price":price}
                else:
                    bid[price]={"share":share,"price":price}
        #### if the message type is cancel, locate the order ID and then subtract the amount for the same share.
        else:
            order_id=data["order_id"][i]
            share=int(data["share"][i][0:3])
            if order_id in order_history.keys():
                if order_history[order_id]["bs_ind"]=="S":
                    price=order_history[order_id]["price"]
                    before_share=ask[price]["share"]
                    before_price=ask[price]["price"]
                    ask[price]={"share":before_share-share,"price":price}
                elif  order_history[order_id]["bs_ind"]=="B":
                    price=order_history[order_id]["price"]
                    before_share=bid[price]["share"]
                    before_price=bid[price]["price"]
                    bid[price]={"share":before_share-share,"price":price}
            else:
                pass
    best_bid_price =[]
    best_bid_size=[]
    best_ask_price=[]
    best_ask_size=[]
    
    #### concatenate the bid, ask price and size together
    for i in range(0,max(len(ask),len(bid))):
             best_bid_price.append("" if (i >= len(bid)) else pd.Series(bid).values.tolist()[i]["price"]  )
             best_bid_size.append( "" if (i >= len(bid)) else pd.Series(bid).values.tolist()[i]["share"])
             best_ask_price.append( "" if (i >= len(ask)) else pd.Series(ask).values.tolist()[i]["price"])
             best_ask_size.append( "" if (i >= len(ask)) else pd.Series(ask).values.tolist()[i]["share"])
             
    ### sort the bid price in descending order while sort the ask price in ascending order
    dynamic_book_order = pd.DataFrame({"Best Bid Price": pd.Series(best_bid_price).values,"Best Bid Size":pd.Series(best_bid_size).values
                 ,"Best Ask Price":pd.Series(best_ask_price).values,"Best Ask Size":pd.Series(best_ask_size).values})
                 
    df_bid = dynamic_book_order[["Best Bid Price","Best Bid Size"]].apply(pd.to_numeric).sort_values("Best Bid Price",ascending=False)

    df_ask = dynamic_book_order[["Best Ask Price","Best Ask Size"]].apply(pd.to_numeric).sort_values("Best Ask Price",ascending=True)

    dynamic_book_order = pd.concat([df_bid.reset_index(),df_ask.reset_index()],axis=1).iloc[:,[1,2,4,5]]

return (dynamic_book_order)   
```

