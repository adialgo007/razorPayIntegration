# Integration With Razor Pay

**1.** Need to configure a rest template for integration with Razor Pay. If we are using spring boot framework, we can make a bean of Rest Template (with annotation @Bean) and autowire it at the required places and use it. For better control and flexibility, we need to configure following in application.properties and configure the rest template with those properties.

```
razorPay.connection.request.time.out=5000
razorPay.read.time.out=20000
razorPay.connect.time.out=10000
razorPay.total.per.route=1000
razorPay.total.max=1000
```

  The above configurations are for timeouts and number of connections to a host and total number of connections that can be opened from the connection pool.
  
**2.** To create an order, call the create order API of razor pay with the required params using the rest template configured in 1st. All the request params should be validated before calling the API like amount is > 0, currency ISO code is valid etc. During the call to razor pay API, following cases may arise:

**i.** We got some valid response from razor pay i.e without any exception.

**ii.** We got connect timeout exception.

**iii.** We got read timeout exception.

**iv.** We got some other exception.

For each of the above cases, the decision to be made is described below:

**i.** For valid response, we just check the status of the order and update the order status accordingly whether it is sucess/failed/pending etc.

**ii.**  Connect timeout means we were not able to connect to Razor Pay. In this case, we can call the create order API again. The number of retries in this case might be 2 or some other value depending on the SLA. If we get connect timeout 3 times (this number can be configured) continuously, we can mark the order as failed with some default error code and error message mapped on the application side for connect timeout.

**iii.** Read timeout means the request has reached razor pay, but we couldn't get the response back in the request timeout configured in 1st. In this case, instead of retrying again for create order, we would just fetch order and get the status of the order. We can retry to fetch order 2 times and if we get valid response from razor pay, we would update the order. If we get request timeouts even after 2 retries, we would mark the order as pending. We can configure a scheduled job (like Rundeck) etc which would run every 15 minutes or so. The purpose of the job would be to fetch all the pending orders and get the order status of those pending jobs and update the order status in the DB for that order once we get a valid response from razor pay.

**iv.** There might be some exception like internal server error or something else. Int this case, we need to confirm with the third party (razor pay) whether there are any special HTTP status codes which would be marked as pending. If we get that status code, then we can mark the order as pending, otherwise we can retry to create order 2 more times and then update the status.

The handling of above 4 cases would take care of all types of order whether it is success, failed and pending. The important thing is the order status on our system and on razor pay system should be in sync. It should not be like on our system the order status is failed and it is success on razor pay side. Above 4 cases tries to avoid those scenarios.

**3.** Once we create the order, we would store all the request and response in a separate collection which stores all the interactions with the razor pay. This collection would be useful for reconciliation between our system and razor pay system.

**4.** After updating the order, we should update the order collection in the DB as well.

**5.** The create order API may be called in a separate thread and the UI might be polling to get order status from an API exposed by the system. The UI can stop polling after sometime like 2 minutes and if the order status is still pending, just mark the order as pending on UI side.

**6.** Payment info can be stored in Order collection only rather than having a separate payment details if we are using NoSQL DB. Or if using SQL DB, we can store payment info in a separate table and put order id in payment table as foreign key on payment table. Similarly, put payment ID as foreign key in Order table.

**7.** The minimal Order Collection may contain following info:

```
unique OrderId
logonId (for e.g email or phone number)
order date 
order status
Items bought
Payment Details
total order amount
applied promo/coupon codes
Response from Razor Pay for creating the order
```

The minimal Payment Details may store following info:

```
unique payment Id
payment method (for e.g. credit card, debit card or UPI)
payment description (for e.g. SBI debit card)
payment date and time
total payment amount
payment status
handling fee
order Id 
cardId or UPI Id (whatever applicable to that particular payment method)
```

# Deployment on Cloud
The application can be deployed as Kubernetes pods on Kubernetes nodes. Multiple pods can be spawned depending on the load of the system. Few of the other resources which needs to be deployed which might be compulsory for the application to run:

1. Databse (Repicated ideally for high availability and durability).
2. Cache (We can use distributed cache like redis sentinel for high availability and durability).
3. Consul (To store application properties out of the application).
4. Vault (To store secrets of the application).


