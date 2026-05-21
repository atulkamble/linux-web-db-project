# Remote MySQL + PHP Web Application on Ubuntu VMs (Azure)

## Architecture

```text
Web VM (Apache + PHP)
        |
        |
        |---- Connects to ----> MySQL DB VM
                                 (10.0.2.4)
```

---

# 1. Prerequisites

## Azure Resources

* 2 Ubuntu Linux VMs
* Same VNET/Subnet
* NSG Rules configured
* Public IP for Web VM
* Private IP for DB VM

Example:

| VM        | Role         | IP        |
| --------- | ------------ | --------- |
| webserver | Apache + PHP | Public IP |
| dbserver  | MySQL Server | 10.0.2.4  |

---

# 2. Configure Database VM

SSH into DB VM:

```bash
ssh azureuser@DB_PUBLIC_IP
```

---

# 3. Install MySQL Server

```bash
sudo apt update -y

sudo apt install mysql-server -y
```

Check service:

```bash
sudo systemctl status mysql
```

Start if needed:

```bash
sudo systemctl start mysql

sudo systemctl enable mysql
```

---

# 4. Configure MySQL Remote Access

Edit MySQL config:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Find:

```text
bind-address = 127.0.0.1
```

Replace with:

```text
bind-address = 0.0.0.0
```

Save:

```text
CTRL + O
ENTER
CTRL + X
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

---

# 5. Verify MySQL Listening

```bash
sudo ss -tulpn | grep 3306
```

Expected:

```text
0.0.0.0:3306
```

---

# 6. Create Database

Login MySQL:

```bash
sudo mysql
```

Create database:

```sql
CREATE DATABASE university;
```

Use database:

```sql
USE university;
```

---

# 7. Create Student Table

```sql
CREATE TABLE student (
    student_id INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    age INT,
    gender VARCHAR(10),
    department VARCHAR(50),
    email VARCHAR(100),
    phone VARCHAR(15),
    city VARCHAR(50),
    admission_date DATE
);
```

---

# 8. Insert Sample Data

```sql
INSERT INTO student
(first_name,last_name,age,gender,department,email,phone,city,admission_date)
VALUES
('Atul','Kamble',24,'Male','Computer Science','atul@example.com','9876543210','Pune','2026-05-21'),

('Ravi','Sharma',22,'Male','Mechanical','ravi@example.com','9876501234','Mumbai','2026-05-20'),

('Sneha','Patil',23,'Female','Electronics','sneha@example.com','9876512345','Nagpur','2026-05-19');
```

Verify:

```sql
SELECT * FROM student;
```

---

# 9. Create Remote MySQL User

```sql
CREATE USER 'studentuser'@'%' IDENTIFIED BY 'StrongPassword123';

GRANT ALL PRIVILEGES ON university.* TO 'studentuser'@'%';

FLUSH PRIVILEGES;
```

Verify users:

```sql
SELECT user,host FROM mysql.user;
```

Exit:

```sql
exit
```

---

# 10. Azure NSG Rule for DB VM

Go to:

Azure Portal → DB VM → Networking

Add Inbound Rule:

| Setting          | Value          |
| ---------------- | -------------- |
| Source           | VirtualNetwork |
| Destination Port | 3306           |
| Protocol         | TCP            |
| Action           | Allow          |
| Priority         | 100            |

---

# 11. Configure Web VM

SSH into Web VM:

```bash
ssh azureuser@WEB_PUBLIC_IP
```

---

# 12. Install Apache + PHP

```bash
sudo apt update -y

sudo apt install apache2 php libapache2-mod-php php-mysql mysql-client -y
```

Start Apache:

```bash
sudo systemctl start apache2

sudo systemctl enable apache2
```

---

# 13. Test MySQL Connectivity

From Web VM:

```bash
nc -zv 10.0.2.4 3306
```

Expected:

```text
Connection to 10.0.2.4 3306 port [tcp/mysql] succeeded!
```

---

# 14. Test MySQL Login

```bash
mysql -h 10.0.2.4 -u studentuser -p
```

Password:

```text
StrongPassword123
```

Run:

```sql
USE university;

SELECT * FROM student;
```

Exit:

```sql
exit
```

---

# 15. Create PHP Database Connection

Go to web directory:

```bash
cd /var/www/html
```

Create db.php:

```bash
sudo nano db.php
```

Add:

```php
<?php

$host = "10.0.2.4";
$user = "studentuser";
$password = "StrongPassword123";
$database = "university";

$conn = new mysqli($host, $user, $password, $database);

if ($conn->connect_error) {
    die("Connection Failed: " . $conn->connect_error);
}

?>
```

---

# 16. Create Web Application

Create index.php:

```bash
sudo nano index.php
```

Add:

```php
<?php
include 'db.php';

$sql = "SELECT * FROM student";
$result = $conn->query($sql);
?>

<!DOCTYPE html>
<html>
<head>

<title>University Student Database</title>

<style>

body{
    font-family: Arial;
    background:#f4f4f4;
    padding:40px;
}

h1{
    text-align:center;
    color:#333;
}

table{
    width:100%;
    border-collapse:collapse;
    background:white;
}

th, td{
    padding:12px;
    border:1px solid #ddd;
    text-align:center;
}

th{
    background:#007bff;
    color:white;
}

tr:nth-child(even){
    background:#f2f2f2;
}

</style>

</head>

<body>

<h1>University Student Details</h1>

<table>

<tr>
    <th>ID</th>
    <th>First Name</th>
    <th>Last Name</th>
    <th>Age</th>
    <th>Gender</th>
    <th>Department</th>
    <th>Email</th>
    <th>Phone</th>
    <th>City</th>
    <th>Admission Date</th>
</tr>

<?php

if ($result->num_rows > 0) {

    while($row = $result->fetch_assoc()) {

        echo "<tr>
                <td>".$row["student_id"]."</td>
                <td>".$row["first_name"]."</td>
                <td>".$row["last_name"]."</td>
                <td>".$row["age"]."</td>
                <td>".$row["gender"]."</td>
                <td>".$row["department"]."</td>
                <td>".$row["email"]."</td>
                <td>".$row["phone"]."</td>
                <td>".$row["city"]."</td>
                <td>".$row["admission_date"]."</td>
              </tr>";
    }

} else {

    echo "<tr><td colspan='10'>No Records Found</td></tr>";

}

?>

</table>

</body>
</html>
```

---

# 17. Restart Apache

```bash
sudo systemctl restart apache2
```

---

# 18. Open Website

```text
http://WEB_VM_PUBLIC_IP/
```

Example:

```text
http://20.xx.xx.xx/
```

---

# 19. Verify Apache

```bash
sudo systemctl status apache2
```

---

# 20. Verify MySQL

```bash
sudo systemctl status mysql
```

---

# 21. Delete Student Entry

Login MySQL:

```bash
sudo mysql
```

Use DB:

```sql
USE university;
```

View records:

```sql
SELECT * FROM student;
```

Delete example:

```sql
DELETE FROM student
WHERE student_id = 2;
```

Verify:

```sql
SELECT * FROM student;
```

Exit:

```sql
exit
```

---

# 22. Troubleshooting

## Check Apache Logs

```bash
sudo tail -f /var/log/apache2/error.log
```

---

## Check MySQL Port

```bash
sudo ss -tulpn | grep 3306
```

---

## Check Connectivity

```bash
nc -zv 10.0.2.4 3306
```

---

## Check Apache Status

```bash
sudo systemctl status apache2
```

---

## Check MySQL Status

```bash
sudo systemctl status mysql
```

---

# Final Output

The PHP webpage dynamically fetches student records from remote MySQL Database VM and displays them through Apache Web Server on Ubuntu.
