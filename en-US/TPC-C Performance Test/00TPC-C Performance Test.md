The TPC-C test is a specification for business processing systems established by the TPC (Transaction Processing Performance Council) committee. The test results mainly depend on traffic metrics and cost-effectiveness metrics and is the most commonly used online transaction processing (OLTP) test benchmark.

The TPC-C test simulates the transaction business scenario of a large commodity wholesaler, which produces and sells a total of 100,000 products and has multiple warehouses responsible for different regions. Each warehouse needs to supply goods to 10 distributors, and each distributor needs to serve 3,000 customers. During the TPC-C test on the database, the number of warehouses can be adjusted according to the configuration of the server and database to simulate different pressure scenarios.

The business scenario mainly includes the following five types of transactions:

- Stock-Level: Used to indicate the inventory status of distributors, which requires replenishment when stock is low.
- New-Order: Used to indicate that a customer has submitted a new order, with each order averaging 10 products.
- Payment: Used to indicate that a customer has paid for an order and updates their account balance.
- Delivery: Used to indicate that goods are shipped according to user-submitted orders.
- Order-Status: Used to query the recent transaction status of users.

This chapter will introduce the TPC-C performance indicators, testing procedures, and optimization approaches of YashanDB on different hardware and software platforms.