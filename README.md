# Nginx Proxy Manager Setup

This repository contains a Docker Compose setup for Nginx Proxy Manager, a web-based nginx proxy management tool.

## ⚠️ Security Notice

This repository has been configured for public sharing. All sensitive data has been removed including:

- Environment files (`.env`)
- Database data (`mysql/`)
- Application data (`data/`)
- SSL certificates and keys
- Log files
- Backup files

## Setup Instructions

### 1. Clone the Repository

```bash
git clone <your-repo-url>
cd nginx-proxy-manager
```

### 2. Create Environment File

Create a `.env` file in the root directory with your own secure credentials:

```bash
# .env
NPM_DB_PASSWORD=your-secure-database-password
NPM_ROOT_PASSWORD=your-secure-npm-admin-password
```

**Important:** Use strong, unique passwords. Never commit the `.env` file to version control.

### 3. Directory Structure

The following directories will be created automatically when you run the containers:

- `data/` - Contains Nginx Proxy Manager application data, configurations, and logs
- `mysql/` - Contains MySQL database data
- `letsencrypt/` - Contains SSL certificates (if using Let's Encrypt)

### 4. Start the Services

```bash
docker-compose up -d
```

### 5. Access the Web Interface

Open your browser and navigate to:
- **HTTP:** `http://your-server-ip:81`
- **HTTPS:** `https://your-server-ip:443` (after SSL setup)

### Default Login Credentials

On first run, use these default credentials (change immediately after login):
- **Email:** `admin@example.com`
- **Password:** `changeme`

## Configuration

### Docker Compose

The `docker-compose.yml` file includes:
- Nginx Proxy Manager web interface
- MySQL database
- Persistent volume mounts for data and database

### Customization

You can customize the setup by:
1. Modifying `docker-compose.yml` for your specific needs
2. Adjusting port mappings if needed
3. Setting up additional environment variables

## Backup and Restore

### Backup

To backup your configuration:

```bash
# Backup application data
sudo tar -czf npm-data-backup.tar.gz data/

# Backup database
docker-compose exec db mysqldump -u npm -p npm > npm-database-backup.sql
```

### Restore

To restore from backup:

```bash
# Restore application data
sudo tar -xzf npm-data-backup.tar.gz

# Restore database
docker-compose exec -T db mysql -u npm -p npm < npm-database-backup.sql
```

## Security Best Practices

1. **Change Default Credentials:** Always change the default admin credentials on first login
2. **Use Strong Passwords:** Set strong passwords in your `.env` file
3. **Keep Updated:** Regularly update the Docker images
4. **Firewall:** Configure firewall rules to restrict access as needed
5. **SSL Certificates:** Use SSL certificates for all proxy hosts
6. **Regular Backups:** Maintain regular backups of your configuration and data

## Troubleshooting

### Common Issues

1. **Port Conflicts:** Ensure ports 80, 81, and 443 are not used by other services
2. **Permission Issues:** The `data/` and `mysql/` directories may need proper permissions
3. **Database Connection:** Check that the MySQL container is running and credentials are correct

### Logs

Check container logs:

```bash
# All services
docker-compose logs

# Specific service
docker-compose logs app
docker-compose logs db
```

## Contributing

If you find issues or have improvements, please feel free to submit a pull request or open an issue.

## License

This setup is provided as-is for educational and practical use. Please ensure compliance with all relevant software licenses.
