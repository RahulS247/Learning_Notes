## Step-by-Step Guide: Deploying Your Application from GitHub to AWS EC2

This guide will walk you through deploying a web application from a GitHub repository to an AWS EC2 instance, including domain configuration and enabling HTTPS.

### 1. Creating an AWS EC2 Instance

1. **Log into AWS Console** and navigate to the **EC2 Dashboard**.
2. Click on **"Launch Instance"**.
3. Select your preferred **Operating System**, recommended: **Ubuntu Server**.
4. Choose an **Instance Type** based on your performance requirements (CPU, RAM).
5. Configure storage (e.g., 20GB is typically sufficient for basic applications).
6. Configure **security groups** (allow HTTP and HTTPS traffic).
7. Generate a new **key pair (.pem)** file, download, and securely store it.
8. Launch the instance. AWS will provide a public IP address (note it down).

### 2. Connecting to Your EC2 Instance

1. Adjust the permissions of your `.pem` file:

   ```bash
   chmod 400 your-key.pem
   ```

2. Connect to the instance using SSH:

   ```bash
   ssh -i /path/to/your-key.pem ubuntu@your-ec2-public-ip
   ```

**Note:** EC2 public IP addresses change on restart. Use an Elastic IP for a static address.

### 3. Setting Up the Server Environment

1. **Update your server** packages:

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Install necessary dependencies**:

   ```bash
   sudo apt install git python3 python3-pip python3-venv nginx -y
   ```

3. **Clone your GitHub repository**:

   ```bash
   git clone https://github.com/your-username/your-repo.git
   ```

4. **Create a Python virtual environment and activate it**:

   ```bash
   python3 -m venv env
   source env/bin/activate
   ```

5. **Install Python dependencies**:

   ```bash
   pip install -r requirements.txt
   ```

### 4. Running the Application

- Start your application using your preferred method (e.g., gunicorn, FastAPI, Django).

Example with FastAPI:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

### 5. Verify Your Application

- Check if your application is running correctly:
  ```bash
  curl http://localhost:8000
  ```

### 6. Setting Up a Domain with AWS Route 53

1. Go to **AWS Route 53**, where you can **register** a new domain (note: this requires a payment), or use an existing domain you already own.
2. Navigate to **"Hosted Zones"** and select your domain.
3. **Create a new A record**:
   - Record type: **A**
   - Route traffic to your EC2 public IP address.
4. Optionally, create a **CNAME record** for subdomains (e.g., `www.yourdomain.com`).

### 7. Securing Your Application with HTTPS (SSL)

1. Navigate to **AWS Certificate Manager (ACM)**:
   - Request a new public SSL certificate. Be sure to add both the root domain (e.g., yourdomain.com) and the 'www' subdomain (e.g., [www.yourdomain.com](http://www.yourdomain.com)) so that users can access your application from either version.
   - Choose DNS validation.
   - After requesting the certificate, click on "Add record to Route 53" to automatically insert the necessary DNS validation record. This links your certificate to your domain name and is required to prove domain ownership.

### 8. Configure Nginx to Use SSL Certificates

Nginx is a high-performance, reliable web server and reverse proxy server commonly used for routing traffic to web applications and handling SSL/TLS encryption. It receives public traffic on ports 80 (HTTP) and 443 (HTTPS), then forwards it internally to your application server running on port 8000 (e.g., Uvicorn for FastAPI). This setup helps separate concerns between web traffic handling and application logic. Alternatives to Nginx include Apache (more traditional, modular) and Caddy (modern, HTTPS-by-default).

1. Install Certbot for automatic SSL handling:

   ```bash
   sudo apt install certbot python3-certbot-nginx -y
   ```

2. Obtain SSL certificate using Certbot:

   ```bash
   sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
   ```

This command typically configures Nginx automatically to use the SSL certificates. Note: you get the `fullchain.pem` and `privkey.pem` files only when you generate an SSL certificate directly on your EC2 server using a tool like Certbot with Let's Encrypt. If Certbot does not configure your Nginx automatically:

- Manually edit your Nginx configuration file, usually located at `/etc/nginx/sites-available/yourdomain.com` or `/etc/nginx/sites-enabled/default`:

  ```bash
  sudo nano /etc/nginx/sites-enabled/yourdomain.com
  ```

- Add SSL paths clearly, like below:

  ```nginx
  server {
      listen 443 ssl;
      server_name yourdomain.com www.yourdomain.com;

      ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

      location / {
          proxy_pass http://localhost:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
  }
  ```

- Save changes and reload Nginx:

  ```bash
  sudo systemctl reload nginx
  ```

After correctly setting these paths, you should see a success message like:

"Successfully deployed certificate for yourdomain.com. Congratulations! You have successfully enabled HTTPS on [https://yourdomain.com."###](https://yourdomain.com."###) 9. Redirecting HTTP to HTTPS

Ensure all traffic redirects from HTTP to HTTPS:

- Edit the Nginx configuration:

  ```bash
  sudo nano /etc/nginx/sites-available/default
  ```

- Ensure these settings exist within your server block:

  ```nginx
  server {
      listen 80;
      server_name yourdomain.com www.yourdomain.com;
      return 301 https://$host$request_uri;
  }

  server {
      listen 443 ssl;
      server_name yourdomain.com www.yourdomain.com;

      ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

      location / {
          proxy_pass http://localhost:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
  }
  ```

- Reload Nginx:

  ```bash
  sudo systemctl reload nginx
  ```

### 10. Final Verification

- Check the HTTP and HTTPS status:
  ```bash
  curl -I http://yourdomain.com
  curl -I https://yourdomain.com
  ```

You should see a **200 OK** status and proper HSTS headers.

### Conclusion

You have successfully deployed your application securely from GitHub to AWS EC2, complete with domain registration and SSL setup.

