# Client Responder

This is an end to end pipeline that monitors gmail for customer message, extracts the necessary details and creates an task in Monday.com task manager.

## Data Loader Pipeline

1. The data loader pipeline fetches all `https://dummyjson.com/products?limit=0` using this api. The `limit=0` parameter removes pagination and fetched all 194 available products.

2. After fetching the products It saves the product details in Google Sheet

## Email to Task Creation Pipeline

1. Monitors Gmail for any unread emails
2. sends the email address, subject and body to an gemini model 2.5 flash, to get order information in json format. Example:

```json
[
  {
    "title": "Urgent Order for Red T-Shirt (ID 42)",
    "date": "10/10/2025, 00:00 AM",
    "description": "Customer John Doe (Phone: +880123456789, Address: 45 Park Street, Dhaka) has placed an urgent order for 3 units of Red T-Shirt (Product ID: 42). Delivery is requested by October 10, 2025. Please confirm once this is scheduled.",
    "product_quantity": "3",
    "product_reference": "42"
  }
]
```

3. Then uses product_reference which can be product id or product name looks up at products database in google sheet to find relevant product.

Example Product response from Google Sheet:

```json
[
  {
    "row_number": 43,
    "id": 42,
    "name": "Water",
    "price": 0.99,
    "category": "groceries"
  }
]
```

4. Then information from previous 2 nodes is integrated and send to another
   gemini model to create necessary details of task. Then extract the data in json model:

```json
[
  {
    "task_title": "Urgent Order Discrepancy: ID 42 (Requested Red T-Shirt, Found Water)",
    "task_description": "Customer John Doe (Phone: +880123456789, Address: 45 Park Street, Dhaka) has placed an urgent order for 3 units.\n\nRequested Product: Red T-Shirt (Product ID: 42)\nSystem Lookup for ID 42: Water (Category: groceries, Price: 0.99)\n\nThere is a discrepancy between the customer's requested product (Red T-Shirt) and the product found in the system for ID 42 (Water).\n\nDelivery is requested by October 10, 2025.\n\nPlease investigate this product discrepancy and confirm the correct item before scheduling delivery. Clarification with the customer may be needed.",
    "customer_name": "John Doe",
    "customer_contact": "+880123456789",
    "delivery_date": "2025-10-10",
    "product_id": 42,
    "product_name": "Water",
    "product_category": "groceries",
    "product_price": 0.99,
    "priority": "High",
    "product_quantity": 3
  }
]
```

5. After task is formatted it is sent to Monday.com to create a task in Tasks boards Customer Order Tables

## Error Handling

- For error handling in HTTP Request fetch Node, 2 Messaging gemini model Nodes that depends on network call made sure they retry 3 times after failure to complete the workflow
