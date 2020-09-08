# Integration With Razor Pay

1. Need to configure a rest template for integration with Razor Pay. If using spring boot framework, we can make a bean of Rest Template (with annotation @Bean) and autowire it at the required places and use it. Also, for better control and flexibility, we need to configure following in application.properties and configure the rest template with those properties.

```
razorPay.connection.request.time.out=5000
razorPay.read.time.out=20000
razorPay.connect.time.out=10000
razorPay.total.per.route=1000
razorPay.total.max=1000
```

  So, the configuration are mostly for timeouts and number of connections to a host and total number of connections that can be opened.
  
2. To pay an order, call the pay order API of razor pay with the required params and using the rest template configured in 1st. All the request params should be validated before calling the API like amount is > 0, currency ISO code is valid etc. During the call to razor pay API, following cases may arise:

i. We got some valid response from razor pay i.e without any exception.

ii. We got connect timeout exception.

iii. We got read timeout exception.

iv. We got some other exception.

For each of the above cases, the decision to be made is described below:

i. For valid response, we just check the status of the order and update the order status accordingly whether it is sucess/failed/pending etc.

ii.  Connect timeout means we were not able to connect to Razor Pay. In this case, we can call the razor pay API again. The number of retries in this case might be 2 or some other value depending on the SLA. So, if we get connect timeout 3 times continuously, we can mark the order as failed with some default error code and error message mapped on the application side for connect timeout.

iii. Read timeout means the request has reached razor pay, but we couldn't get the response back in the request timeout configured. So, in this case, instead of retrying again for create order, we would just fetch order and get the status of the order. We can retry to fetch order 2 times and if we get valid response from razor pay, we would update the order. If even after 2 retries, we get request timeouts, we would mark the order as pending. We can configure a scheduled job (like Rundeck) etc which would run every 15 minutes or so. The purpose of the job would be to fetch all the pending orders and get the order status of those pending jobs and update the order status in the DB for that order once we get a valid response from razor pay.

iv. This might be some exception like internal server error or something else. Int his case, need to confirm with the third party (razor pay) whether there are any special HTTP status codes which would be marked as pending. If we get that status code, then we can mark the order as pending, otherwise we wcan retry to create order 2 more times and then update the status.

3. Once the order is created, we would store all the request and response in a separate collection which stores all the interactions with the razor pay. This collection would be useful for reconciliation.

4. After updating the order, we should update the order collection in DB as well.

5. The pay order API may be called in a separate thread and the UI might be polling to get order status from an API exposed by the system. The UI can stop polling after sometime like 2 minutes and if the order status is still pending, just mark the order as pending on UI side.


