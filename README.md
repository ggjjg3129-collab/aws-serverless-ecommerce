# AWS Serverless E-Commerce Web Application

## 1. Overview

This project is a serverless e-commerce web application built to demonstrate practical AWS cloud architecture and service integration.

The frontend is developed using HTML5, CSS3, and JavaScript, while the backend uses AWS Lambda functions exposed through Amazon API Gateway. The application integrates Amazon RDS for MySQL for persistent data storage and AWS Secrets Manager for secure database credential management.

The project demonstrates how multiple AWS services can work together to provide a secure and scalable backend without managing traditional application servers.

---

## 2. Architecture

![AWS Serverless E-Commerce Architecture](screenshots/architecture-diagram.png)

The application follows a serverless backend architecture:

```text
Frontend
    |
    v
Amazon API Gateway
    |
    v
AWS Lambda
    |
    +----------> AWS Secrets Manager
    |
    v
Amazon RDS for MySQL
```

The backend resources are connected through Amazon VPC networking. Amazon RDS is not publicly accessible, while Lambda is configured to access resources within the VPC.

A VPC Endpoint for Secrets Manager allows Lambda to retrieve database credentials through private networking.

Security Groups control the required communication between Lambda and RDS.

---

## 3. AWS Services Used

### AWS Lambda

AWS Lambda is the primary backend compute service used in this project.

Three Lambda functions handle the main backend operations:

* `Ecommerce-Register`
* `Ecommerce-Login`
* `Ecommerce-Get-Products`

Lambda executes the application logic in response to API requests without requiring server provisioning or server management.

**Practical role:** Execute backend application logic and communicate with the database.

---

### Amazon API Gateway

Amazon API Gateway exposes the Lambda functions through HTTP API endpoints used by the frontend.

The API includes:

```text
POST /register
POST /login
GET /products
```

**Practical role:** Receive frontend requests and route them to the appropriate Lambda function.

---

### Amazon RDS for MySQL

Amazon RDS for MySQL provides the relational database used by the application.

Database:

```text
ecommerce_db
```

Tables:

```text
users
products
```

The `users` table stores registered user information, while the `products` table stores the products displayed in the store.

The database is configured with:

```text
Public Access: No
```

This keeps the database private inside the AWS networking environment.

**Practical role:** Store and retrieve persistent application data.

---

### AWS Secrets Manager

AWS Secrets Manager stores the database credentials required by the Lambda functions.

Lambda retrieves the credentials at runtime instead of storing them directly in the frontend or source code.

**Practical role:** Securely manage sensitive database credentials.

---

### Amazon VPC

Amazon VPC provides private networking for the backend resources.

Amazon RDS is deployed within the VPC, while Lambda is configured to access resources within the VPC. This allows the application to communicate with the database through controlled private networking without exposing the database directly to the public internet.

**Practical role:** Provide isolated and controlled networking for backend resources.

---

### Security Groups

Security Groups control network traffic between the backend resources.

The RDS Security Group allows MySQL traffic on port `3306` from the Lambda Security Group.

```text
Lambda Security Group
        |
        | TCP 3306
        v
RDS Security Group
```

**Practical role:** Restrict database access to the required backend resource.

---

### VPC Endpoint for Secrets Manager

A VPC Endpoint for Secrets Manager allows Lambda to communicate with Secrets Manager privately from inside the VPC.

**Practical role:** Provide private access to Secrets Manager without requiring public internet access.

---

### PyMySQL Lambda Layer

The PyMySQL Lambda layer provides the Python database driver required for Lambda to connect to Amazon RDS for MySQL.

**Practical role:** Enable Python Lambda functions to communicate with the MySQL database.

---

## 4. Application Flow

### User Registration

```text
Frontend
   ↓
API Gateway
   ↓
Ecommerce-Register Lambda
   ↓
Secrets Manager
   ↓
Amazon RDS MySQL
```

When a user registers, the frontend sends the account information to API Gateway.

The Register Lambda retrieves the database credentials from Secrets Manager, hashes the password, and stores the user information in the `users` table in Amazon RDS.

---

### User Login

```text
Frontend
   ↓
API Gateway
   ↓
Ecommerce-Login Lambda
   ↓
Secrets Manager
   ↓
Amazon RDS MySQL
```

The Login Lambda retrieves the database credentials, searches for the user by email, verifies the stored password hash, and returns the login result to the frontend.

---

### Product Retrieval

```text
Frontend
   ↓
API Gateway
   ↓
Ecommerce-Get-Products Lambda
   ↓
Secrets Manager
   ↓
Amazon RDS MySQL
   ↓
Lambda
   ↓
API Gateway
   ↓
Frontend
```

The Get Products Lambda queries the `products` table and returns the product information to the frontend.

The retrieved products are then displayed dynamically in the store.

---

## 5. Database

The application uses Amazon RDS for MySQL with the following database:

```text
Database: ecommerce_db
```

### Users Table

The `users` table stores registered users.

Main fields include:

```text
id
name
email
password_hash
```

Passwords are hashed before being stored in the database.

### Products Table

The `products` table stores the products displayed by the application.

Main fields include:

```text
id
name
description
price
stock
created_at
```

The project includes the following sample products:

```text
Premium T-Shirt
Classic Hoodie
Sport Cap
Sports Bag
```

---

## 6. Lambda Functions

### 6.1 Ecommerce-Register

The Register Lambda handles user registration.

Responsibilities:

* Retrieve database credentials from Secrets Manager
* Receive registration data
* Hash the user's password
* Insert the user into the RDS database
* Return the registration result

```python
import json
import boto3
import pymysql
import hashlib
import os

SECRET_NAME = "rds!db-e3663d17-6fe4-4488-9454-a42c3f79f307"
REGION = "us-east-1"

RDS_HOST = "ecommerce-db.cg7gmm0quohu.us-east-1.rds.amazonaws.com"
RDS_PORT = 3306

def hash_password(password):
    salt = os.urandom(16)
    password_hash = hashlib.scrypt(
        password.encode(),
        salt=salt,
        n=16384,
        r=8,
        p=1
    )
    return salt.hex() + ":" + password_hash.hex()

def lambda_handler(event, context):
    secrets_client = boto3.client(
        "secretsmanager",
        region_name=REGION
    )

    response = secrets_client.get_secret_value(
        SecretId=SECRET_NAME
    )

    secret = json.loads(response["SecretString"])

    body = json.loads(event["body"])

    name = body["name"]
    email = body["email"]
    password = body["password"]

    password_hash = hash_password(password)

    connection = pymysql.connect(
        host=RDS_HOST,
        user=secret["username"],
        password=secret["password"],
        port=RDS_PORT,
        database="ecommerce_db",
        connect_timeout=5
    )

    with connection.cursor() as cursor:
        cursor.execute(
            """
            INSERT INTO users (name, email, password_hash)
            VALUES (%s, %s, %s)
            """,
            (name, email, password_hash)
        )

    connection.commit()
    connection.close()

    return {
        "statusCode": 201,
        "body": json.dumps({
            "message": "User registered successfully"
        })
    }
```

---

### 6.2 Ecommerce-Login

The Login Lambda authenticates users against the data stored in RDS.

Responsibilities:

* Retrieve database credentials
* Find the user by email
* Retrieve the stored password hash
* Verify the submitted password
* Return the authenticated user information

```python
import json
import boto3
import pymysql
import hashlib
import hmac

SECRET_NAME = "rds!db-e3663d17-6fe4-4488-9454-a42c3f79f307"
REGION = "us-east-1"

RDS_HOST = "ecommerce-db.cg7gmm0quohu.us-east-1.rds.amazonaws.com"
RDS_PORT = 3306

def verify_password(password, stored_hash):
    salt_hex, hash_hex = stored_hash.split(":")
    salt = bytes.fromhex(salt_hex)
    original_hash = bytes.fromhex(hash_hex)

    new_hash = hashlib.scrypt(
        password.encode(),
        salt=salt,
        n=16384,
        r=8,
        p=1
    )

    return hmac.compare_digest(new_hash, original_hash)

def lambda_handler(event, context):

    secrets_client = boto3.client(
        "secretsmanager",
        region_name=REGION
    )

    response = secrets_client.get_secret_value(
        SecretId=SECRET_NAME
    )

    secret = json.loads(response["SecretString"])

    body = json.loads(event["body"])

    email = body["email"]
    password = body["password"]

    connection = pymysql.connect(
        host=RDS_HOST,
        user=secret["username"],
        password=secret["password"],
        port=RDS_PORT,
        database="ecommerce_db",
        connect_timeout=5
    )

    with connection.cursor() as cursor:
        cursor.execute(
            """
            SELECT id, name, email, password_hash
            FROM users
            WHERE email = %s
            """,
            (email,)
        )

        user = cursor.fetchone()

    connection.close()

    if user is None:
        return {
            "statusCode": 401,
            "body": json.dumps({
                "message": "Invalid email or password"
            })
        }

    user_id, name, user_email, stored_hash = user

    password_valid = verify_password(
        password,
        stored_hash
    )

    if not password_valid:
        return {
            "statusCode": 401,
            "body": json.dumps({
                "message": "Invalid email or password"
            })
        }

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "Login successful",
            "user": {
                "id": user_id,
                "name": name,
                "email": user_email
            }
        })
    }
```

---

### 6.3 Ecommerce-Get-Products

The Get Products Lambda retrieves product data from RDS and returns it to the frontend.

```python
import json
import boto3
import pymysql

SECRET_NAME = "rds!db-e3663d17-6fe4-4488-9454-a42c3f79f307"
REGION = "us-east-1"

RDS_HOST = "ecommerce-db.cg7gmm0quohu.us-east-1.rds.amazonaws.com"
RDS_PORT = 3306

def lambda_handler(event, context):

    secrets_client = boto3.client(
        "secretsmanager",
        region_name=REGION
    )

    response = secrets_client.get_secret_value(
        SecretId=SECRET_NAME
    )

    secret = json.loads(response["SecretString"])

    connection = pymysql.connect(
        host=RDS_HOST,
        user=secret["username"],
        password=secret["password"],
        port=RDS_PORT,
        database="ecommerce_db",
        connect_timeout=5
    )

    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT id, name, description, price, stock
            FROM products
            ORDER BY id
        """)

        products = cursor.fetchall()

    connection.close()

    result = []

    for product in products:
        result.append({
            "id": product[0],
            "name": product[1],
            "description": product[2],
            "price": float(product[3]),
            "stock": product[4]
        })

    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "body": json.dumps({
            "products": result
        })
    }
```

---

## 7. Lambda Test Result

The `Ecommerce-Get-Products` Lambda was tested successfully.

The Lambda returned an HTTP `200` response and retrieved the product records stored in Amazon RDS.

json
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"products\": [{\"id\": 1, \"name\": \"Premium T-Shirt\", \"description\": \"High-quality comfortable t-shirt\", \"price\": 25.0, \"stock\": 100}, {\"id\": 2, \"name\": \"Classic Hoodie\", \"description\": \"Classic comfortable hoodie\", \"price\": 45.0, \"stock\": 50}, {\"id\": 3, \"name\": \"Sport Cap\", \"description\": \"Lightweight sports cap\", \"price\": 18.0, \"stock\": 75}, {\"id\": 4, \"name\": \"Sports Bag\", \"description\": \"Durable sports bag for training\", \"price\": 35.0, \"stock\": 40}]}"
}
```

This confirms that Lambda successfully connected to the MySQL database, queried the `products` table, and returned the stored records.

---



## 9. Security and Networking

The application uses several AWS security and networking controls:

* Amazon RDS is not publicly accessible.
* Lambda accesses RDS through the VPC.
* Security Groups restrict MySQL access to the required resources.
* Database credentials are stored in AWS Secrets Manager.
* Credentials are not exposed in frontend code.
* Passwords are hashed before being stored.
* Lambda accesses Secrets Manager through a VPC Endpoint.
* RDS accepts MySQL traffic on port `3306` from the Lambda Security Group.

These configurations provide controlled communication between the application components while keeping the database private.

---

## 10. Frontend

The frontend is built using:

* HTML5
* CSS3
* JavaScript

The application provides:

* Home page
* User registration
* User login
* Logout
* Welcome message
* Dynamic product loading
* Shopping cart
* Quantity controls
* Dark and light mode
* Responsive design

The product data displayed on the Store page is retrieved dynamically from the AWS backend.

---

## 11. Project Structure

```text
Lambda-Project/
├── index.html
├── login.html
├── register.html
├── store.html
├── css/
│   └── style.css
├── js/
│   └── script.js
├── assets/
│   └── logo.svg
├── screenshots/
│   └── architecture-diagram.png
└── .gitignore
```

---

## 12. Technologies

* HTML5
* CSS3
* JavaScript
* Python
* AWS Lambda
* Amazon API Gateway
* Amazon RDS for MySQL
* AWS Secrets Manager
* Amazon VPC
* Security Groups
* VPC Endpoint
* PyMySQL

---

## 13. What I Learned

This project provided practical experience with:

* Designing a serverless AWS architecture
* Building backend APIs using API Gateway and Lambda
* Connecting Lambda to Amazon RDS MySQL
* Using Secrets Manager for secure database credentials
* Configuring Lambda and RDS inside a VPC
* Using Security Groups to control database access
* Using a VPC Endpoint for private Secrets Manager access
* Building frontend applications that consume AWS APIs
* Working with relational database tables and application data
* Integrating multiple AWS services into a complete application workflow

---

## 14. Future Improvements

The following capabilities could be added in a future version:

* Backend order management
* Checkout API
* Amazon Cognito authentication
* Payment integration
* Amazon S3 and CloudFront deployment
* Infrastructure as Code
* Advanced monitoring and logging

---

## 15. Conclusion

This project demonstrates a practical serverless e-commerce architecture using AWS Lambda as the primary backend compute service.

The application integrates Amazon API Gateway, AWS Lambda, Amazon RDS for MySQL, AWS Secrets Manager, Amazon VPC, Security Groups, and a VPC Endpoint into a working cloud-based application.

The completed workflow demonstrates how frontend requests can be processed by serverless backend functions, securely access a private relational database, and return real application data to the user.
