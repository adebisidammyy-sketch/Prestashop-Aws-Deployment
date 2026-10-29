## PrestaShop Deployment on AWS# #
**Author:** Adebisi Damilola Abiola

A working PrestaShop e-commerce store deployed on AWS using only Free Tier resources. The database runs on a separate server from the web application and is not exposed to the internet.

**Live store:** http://51.21.182.10

**Full write-up with screenshots:** [PrestaShop_AWS_Deployment_Documentation.pdf](./PrestaShop_AWS_Deployment_Documentation.pdf.pdf)

---

## Architecture

| Component | Service | Details |
|---|---|---|
| Web/app server | Amazon EC2 | Ubuntu 26.04 LTS, t3.micro |
| Database server | Amazon RDS for MySQL | db.t3.micro, separate from the EC2 instance |
| Networking | Default VPC (eu-north-1) | Security groups control all access between the two |
| Application | PrestaShop 9.1.5 | Installed with the official web installer |

```
Internet ──(80/443)──> EC2: prestashop-web-sg ──(3306)──> RDS: prestashop-db-sg
                       Apache + PHP 8.5                    MySQL, no public access
```

## Security decisions

- **Database isolation:** RDS has Public Access set to **No**, so it has no public IP and can't be reached from the internet.
- **Least-privilege networking:** the database security group allows port 3306 only from the EC2 security group (`prestashop-web-sg`), not from an IP range.
- **Dedicated security groups:** one for the web server and one for the database, so each can be managed independently.
- **Encryption and backups:** RDS encryption and automated backups are enabled.
- **Installer cleanup:** the `/install` directory was deleted after setup.
- **Randomized admin path:** PrestaShop generated a non-guessable back-office URL.
- **Known improvement:** SSH is open to all sources in this setup. In production it should be restricted to a specific IP, and HTTPS should be enabled.

## Deployment steps

1. **Create the database (RDS):** MySQL engine, Full configuration, Free Tier template, instance `prestashop-db`, initial database `prestashop`, Public Access = No, new security group `prestashop-db-sg`.
2. **Launch the web server (EC2):** Ubuntu 26.04 LTS, t3.micro, new key pair, security group `prestashop-web-sg` with inbound SSH (22), HTTP (80) and HTTPS (443).
3. **Link the two:** add an inbound MySQL/Aurora rule (port 3306) to `prestashop-db-sg` with `prestashop-web-sg` as the source.
4. **Connect to the server:** EC2 Instance Connect from the AWS console.
5. **Install the stack:** Apache2, PHP 8.5 with extensions (mysql, gd, curl, zip, xml, mbstring, intl, bcmath), and unzip.
6. **Install PrestaShop:** download 9.1.5, extract to `/var/www/html/prestashop`, set ownership to `www-data`, and point Apache's DocumentRoot at that folder.
7. **Tune the server:**
   - Raise PHP limits (`max_execution_time`, `memory_limit`, `upload_max_filesize`, `post_max_size`).
   - Add a 1 GB swap file, because the t3.micro's ~900 MB of RAM caused the installer to be killed by the OOM killer.
   - Set `AllowOverride All` for the PrestaShop directory so `.htaccess` rewriting works.
8. **Run the web installer:** point it at the RDS endpoint. "Database is connected" confirmed the security group link worked.
9. **Verify:** storefront loads publicly, and the admin dashboard works with the generated credentials.

## Results

- Storefront is publicly accessible with the default Hummingbird theme.
- Back office is accessible and fully functional.
- Database is hosted separately on RDS with no public exposure.
- Everything runs on Free Tier resources.

## Lessons learned

- A small instance needs swap space to run the PrestaShop installer reliably.
- Referencing a security group as the source, instead of an IP, makes the database rule tighter and easier to maintain.
- Choosing "Full configuration" in RDS gives control over public access and security groups that "Easy create" hides.

## Repository contents

- `README.md`: this overview
- `PrestaShop_AWS_Deployment_Documentation.pdf`: the complete documentation with screenshots
