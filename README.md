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

