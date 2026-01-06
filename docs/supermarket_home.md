# iCash Supermarket System — Project Brief (Markdown)

## Guidelines

### General Instructions
1. **GitHub**: The code must be managed in a GitHub repository under your ownership.  
2. **Readability**: The code must be clean and readable, utilizing meaningful names and comments.  
3. **Language**: Use **Python**.  
4. **Deployment**: The system must be started and stopped using **docker-compose**.  
5. **AI Tools**: You are permitted to use AI tools to assist in development.  
6. **Submission**: Upon completion, reply via email with:
   - A **Block Diagram** of the system architecture you designed
   - A link to your **Public Repository**

---

## The Project: iCash Supermarket System

iCash produces cash register systems for supermarkets. You are required to build an **interactive micro-services system** based on **Docker** with a **user interface**.

To keep the scope focused, the system will be implemented for a retail chain with the following characteristics:
- **3 Branches** (Locations)
- **10 Products** available for sale
- **Purchase Limit**: Each customer can buy only **one unit of any specific product** per transaction

---

## Data Files

You are provided with data files that must be loaded into the database:
- `Purchases.csv` — Historical purchase data
- `Products_list.csv` — List of products

---

## Version 1.0 Requirements

### 1) Application A: Cash Register Simulator
This application simulates the activity of a cash register. At the end of every purchase, the register reports the following data to the `purchases` table in the database:

- **Supermarket ID**: selection from a predefined list  
- **Timestamp**: current time  
- **User ID (uuid4)**:
  - **New Customer**: generate a new unique ID
  - **Returning Customer**: allow selection from the list of existing customers in the system
- **Items List**: selection of products from the product list  
- **Total Amount**: calculated based on selected items  

---

### 2) Application B: Management Dashboard
This application allows the supermarket owner to retrieve the following **real-time** data:

1. **Unique Customers**: the total count of unique shoppers across the entire chain  
2. **Loyal Customers**: a list of customers who have made at least **3 purchases** in the past  
3. **Top Products**: a list of the **top 3 best-selling products of all time**  
   - If multiple products have the same popularity, include **more than 3**
