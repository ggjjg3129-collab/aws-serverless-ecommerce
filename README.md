# AWS Serverless E-Commerce Web Application

## 1. Overview

This project is a serverless e-commerce web application designed to demonstrate practical AWS integration patterns used in modern cloud-based applications. The frontend is a lightweight web experience built with HTML5, CSS3, and JavaScript, while the backend is implemented using AWS Lambda functions exposed through Amazon API Gateway.

The project focuses on real-world serverless architecture and demonstrates how frontend applications can securely connect to backend services that use managed AWS infrastructure. In particular, it showcases the role of AWS Lambda as the primary backend compute service for processing application logic without managing servers.

## 2. Project Objectives

This project was built to gain hands-on experience with the following AWS and application-development concepts:

* Serverless architecture
* AWS Lambda
* API development with Amazon API Gateway
* Database integration with Amazon RDS for MySQL
* Secure credential management with AWS Secrets Manager
* VPC networking and private service communication
* Security Group configuration
* Connecting a frontend web application to cloud services

The emphasis is on understanding how different AWS services work together to support a real application architecture in a secure, scalable, and operationally efficient manner.

## 3. Architecture

The application follows a simple serverless architecture:

```text
Frontend
    |
    v
Amazon API Gateway
    |
    v
AWS Lambda
    |
    +----> AWS Secrets Manager
    |
    v
Amazon RDS for MySQL
```

The frontend communicates with the backend through API Gateway. AWS Lambda handles the business logic and interacts with Amazon RDS MySQL for data operations. Database credentials are retrieved securely from AWS Secrets Manager. Amazon RDS is deployed within the VPC, while Lambda is configured to access resources in the VPC. Security Groups control the permitted network traffic between them. This architecture demonstrates how Lambda can serve as the application backend while keeping database access private and controlled.

## 4. AWS Services Used

### AWS Lambda

AWS Lambda is the core backend compute service in this project. It executes the application logic without requiring server provisioning or infrastructure maintenance. Lambda functions are used to handle user registration, login, and product retrieval. This demonstrates the practical value of serverless backend compute for web applications, where code runs in response to API requests and scales automatically based on demand.

### Amazon API Gateway

Amazon API Gateway exposes the backend functionality through HTTP endpoints that the frontend calls. It provides a standardized way for the frontend to invoke backend logic while managing API routing, request handling, and secure communication between the client and Lambda.

### Amazon RDS for MySQL

Amazon RDS for MySQL provides a managed relational database service for storing application data. In this project, the backend uses it to store and retrieve user-related and product-related data. Using a managed database reduces operational overhead while providing reliability and compatibility for structured application data.

### AWS Secrets Manager

AWS Secrets Manager securely stores the database credentials used by the application. Lambda retrieves these credentials at runtime instead of embedding them in the frontend or application code. This is a key security practice that reduces the risk of exposing sensitive configuration values.

### Amazon VPC

Amazon VPC enables private networking for the backend resources. Amazon RDS is deployed within the VPC, while Lambda is configured to access resources within the VPC. This allows the application to communicate with the database through controlled private networking without exposing the database directly to the public internet.

### Security Groups

Security Groups are used to control inbound and outbound traffic for the resources in the VPC. They are configured to allow only the required communication paths, such as allowing Lambda access to RDS over the MySQL port while restricting unnecessary access. This improves network isolation and reduces attack surface.

### VPC Endpoint for Secrets Manager

A VPC Endpoint for Secrets Manager allows Lambda to access secrets privately without traversing the public internet. This enables secure credential retrieval inside the VPC and demonstrates a common AWS networking pattern for private service access.

### PyMySQL Lambda Layer

The PyMySQL Lambda layer provides the database driver required for Lambda functions to connect to Amazon RDS MySQL. This allows the Lambda functions to interact with the relational database in a clean and maintainable way, while keeping the application logic focused on API behavior and business rules.

## 5. Application Flow

### Registration

Frontend → API Gateway → Register Lambda → Secrets Manager → RDS MySQL

When a new user registers, the frontend sends account details to the API Gateway endpoint. Lambda retrieves the database credentials from Secrets Manager and stores the user information in Amazon RDS MySQL. Passwords are hashed before insertion to protect stored credentials.

### Login

Frontend → API Gateway → Login Lambda → Secrets Manager → RDS MySQL

During login, the frontend submits credentials to the API Gateway. The login Lambda function retrieves the database credentials securely, queries the database for the user record, verifies the password, and returns the login result to the frontend.

### Products

Frontend → API Gateway → Get Products Lambda → Secrets Manager → RDS MySQL → Lambda → API Gateway → Frontend

The product listing flow shows how Lambda can fetch data from MySQL and return it to the frontend through the API layer. This is a strong example of how serverless backend functions can power dynamic web content while keeping database access private and secure.

## 6. Shopping Cart

The shopping cart is handled entirely in the browser using localStorage. This keeps the cart state available during the user session without requiring a backend order workflow. The cart supports the following client-side behaviors:

* Add products
* Increase product quantity
* Decrease product quantity
* Remove products
* Calculate the total cost

This project does not implement checkout, order processing, or payment handling, and those capabilities remain out of scope for the current application.

## 7. Security and Networking

The application follows several important AWS security and networking practices:

* RDS is not publicly accessible.
* Lambda communicates with RDS through the VPC.
* Security Groups restrict MySQL access to the required resources.
* Database credentials are stored in AWS Secrets Manager.
* Credentials are not exposed in frontend code.
* Passwords are hashed before being stored in the database.
* Secrets Manager is accessed privately through a VPC Endpoint.

These controls demonstrate how cloud-native applications can maintain a secure architecture while still providing the functionality needed by a web frontend.

## 8. Frontend

The frontend is implemented with standard web technologies and provides the user-facing interface for the application:

* HTML5
* CSS3
* JavaScript

The interface includes:

* Registration
* Login
* Logout
* Welcome message
* Dynamic product loading
* Shopping cart with quantity controls
* Dark and light mode toggle
* Responsive design

The frontend is intentionally kept lightweight and focused on user interaction while the backend responsibilities are handled by AWS services.

## 9. Project Structure

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
└── .gitignore
```

This repository contains the frontend application structure used to connect to the AWS backend services described in this project. No additional backend files are included in this repository.

## 10. Technologies

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
* PyMySQL

## 11. What I Learned

This project demonstrates practical experience with the following AWS and application integration concepts:

* Building a serverless backend with AWS Lambda
* Connecting API Gateway to Lambda-based application logic
* Integrating Lambda with Amazon RDS MySQL
* Using AWS Secrets Manager for secure credential management
* Working with VPC networking and private communication patterns
* Configuring Security Groups for resource isolation
* Building APIs that support frontend-driven applications
* Integrating multiple AWS services into a cohesive application architecture
* Understanding how serverless components communicate in a real application environment

## 12. Future Improvements

The following items are planned as future improvements and are not implemented in the current version of this project:

* Backend order management
* Checkout API
* Amazon Cognito authentication
* Payment integration
* S3 and CloudFront deployment
* Infrastructure as Code
* Improved monitoring and logging

## 13. Conclusion

This project demonstrates practical experience in designing and integrating a serverless AWS application with AWS Lambda as the primary backend compute service. It combines frontend interaction, API exposure, secure secret management, database connectivity, and private networking into a working architecture that reflects common patterns used in cloud-native application development.
