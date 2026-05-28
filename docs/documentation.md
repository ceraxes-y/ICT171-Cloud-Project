# 01 - Virtual Machine Setup

## Cloud Provider
Microsoft Azure — Azure for Students (free credits, no credit card required)
Sign up at https://azure.microsoft.com/en-us/free/students

## VM Specifications
| Parameter | Value |
|---|---|
| VM Name | ict171-server |
| Region | East Asia  (Australia East is blocked under Azure for Students) |
| Image | Ubuntu Server 24.04 LTS |
| Size | Standard B2ats v2 (2 vCPUs, 1 GiB RAM) |
| Authentication | SSH public key |
| Username | azureuser |

## Creating the VM
1. In the Azure Portal, go to **Virtual Machines → Create → Azure virtual machine**
2. Fill in the specifications from the table above
3. Under **Authentication**, select **SSH public key** and generate a new key pair named `ict171-key`
4. When prompted, download the private key file `ict171-key.pem` and save it to `~/Downloads`

## Opening Ports
In the VM's **Networking** settings, add inbound rules for:
| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH access |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |

## Connecting to the VM
Set the correct permissions on the key file, then connect:

```bash
chmod 400 ~/Downloads/ict171-key.pem
ssh -i ~/Downloads/ict171-key.pem azureuser@20.2.235.169
```

## First Update
```bash
sudo apt update && sudo apt upgrade -y
```

# 02 - Web Server Setup (Nginx)

## Install Nginx

```bash
sudo apt install nginx -y
```

## Start and enable Nginx
Enable Nginx to start automatically on reboot:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Verify it is running:

```bash
sudo systemctl status nginx
```

## Test the installation
At this point, visiting `http://20.2.235.169` in a browser should display the default Nginx welcome page.

## Configure the web root
The site files are served from `/var/www/html/`. Set the correct permissions so Nginx can read them:

```bash
sudo chmod 755 /var/www/html
sudo chmod 644 /var/www/html/*.html /var/www/html/*.css
```

## Remove the default Nginx page
```bash
sudo rm /var/www/html/index.nginx-debian.html
```

## Verify Nginx config is valid
```bash
sudo nginx -t
```

## Reload Nginx after any config change
```bash
sudo systemctl reload nginx
```
# 03 - Domain Name & SSL Setup

## Domain Registration
Domain `thronelore.site` was registered via Namecheap at https://www.namecheap.com

## Linking the Domain to the VM
In the Namecheap dashboard, go to **Domain List → Manage → Advanced DNS** and add the following records:

| Type | Host | Value | TTL |
|---|---|---|---|
| A Record | @ | 20.2.235.169 | Automatic |
| A Record | www | 20.2.235.169 | Automatic |

Wait a few minutes for DNS propagation. Verify with:

```bash
ping thronelore.site
```

## SSL/TLS with Let's Encrypt

### Install Certbot
```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Obtain and install the certificate
```bash
sudo certbot --nginx -d thronelore.site -d www.thronelore.site
```

Follow the prompts:
- Enter your email address
- Agree to the terms of service
- Certbot will automatically modify the Nginx config to enable HTTPS

### Verify HTTPS
Visit `https://thronelore.site` in a browser — the padlock should appear.

### Auto-renewal
Certbot sets up automatic renewal by default. Test it with:

```bash
sudo certbot renew --dry-run
```
# 04 - Site Deployment

## Getting the Site Files
Clone the repository on your local machine:

```bash
git clone https://github.com/ceraxes-y/ICT171-Cloud-Project.git
```

The site files are located in the `site/` folder of the repository.

## Site Structure
The site consists of 5 files served as static HTML/CSS:

| File | Purpose |
|---|---|
| `index.html` | Homepage |
| `characters.html` | Characters page |
| `houses.html` | Houses & Lore page |
| `timeline.html` | Episodes timeline with spoiler system |
| `style.css` | Shared stylesheet for all pages |
| `images/...` | Houses sigils |

All files are served from `/var/www/html/`
## Transferring Files to the VM
From your local machine, use `scp` to transfer all files at once:

```bash
scp -i ~/Downloads/ict171-key.pem ~/Downloads/files/index.html \
~/Downloads/files/characters.html \
~/Downloads/files/houses.html \
~/Downloads/files/timeline.html \
~/Downloads/files/style.css \
azureuser@20.2.235.169:~/
```

## Transferring the Images Folder
```bash
scp -i ~/Downloads/ict171-key.pem -r ~/Desktop/images azureuser@20.2.235.169:~/
```

## Moving Files to the Web Root
Once transferred, move them from the home directory to `/var/www/html/`:

```bash
sudo mv ~/index.html ~/characters.html ~/houses.html ~/timeline.html ~/style.css /var/www/html/
```

## Setting Correct Permissions

```bash
sudo chmod 644 /var/www/html/*.html /var/www/html/*.css
```

## Verify Deployment
Check all files are in place:

```bash
ls -la /var/www/html/
```


Visit `https://thronelore.site` to confirm the site loads correctly.

# 05 - Spoiler Prevention Script

## Overview
The timeline page (`timeline.html`) includes a JavaScript spoiler 
prevention system. It allows users to select which season they have 
watched up to, and automatically locks all seasons beyond that point 
with a blur filter and a padlock overlay. This prevents users from 
being spoiled on episodes they have not yet seen. The selected season 
is saved in `sessionStorage` so it persists as the user navigates 
between pages.

## Script Location
The script is embedded at the bottom of `/var/www/html/timeline.html`
within a `<script>` tag.

## Code

```javascript
// applySeason: triggered when the user selects a season from the navbar dropdown
// val = the season number the user has watched up to (e.g. "4")
function applySeason(val) {

  // Save the selected season in sessionStorage so it persists across pages
  sessionStorage.setItem('gotSeason', val);

  // Convert the string value to an integer for comparison
  const max = parseInt(val);

  // Select all timeline items on the page
  const items = document.querySelectorAll('.timeline-item');

  // Loop through each timeline item
  items.forEach(item => {

    // Read the season number stored in the data-season attribute
    const s = parseInt(item.getAttribute('data-season'));

    // If the season has been watched, unlock it (remove blur and padlock)
    if (s <= max) {
      item.classList.remove('locked');

    // Otherwise lock it to prevent spoilers
    } else {
      item.classList.add('locked');
    }
  });

  // Update the progress bar width based on how many seasons have been watched
  const fill = document.getElementById('progress-fill');
  const label = document.getElementById('progress-label');
  fill.style.width = (max / 8 * 100) + '%';

  // Update the progress label text
  label.textContent = max === 8 ? 'All 8 seasons' : 'Season ' + max + ' of 8';
}

// On page load: restore the previously selected season from sessionStorage
// If nothing is saved, default to season 8 (all seasons visible)
const saved = sessionStorage.getItem('gotSeason') || '8';
document.getElementById('season-nav').value = saved;
applySeason(saved);
```

## Verification
The script can be tested live at https://thronelore.site/timeline.html — 
select a season from the navbar dropdown and verify that seasons beyond 
that point are blurred with a padlock overlay.

## Updating the Site
To update any file, repeat the `scp` and `mv` steps above for the modified file.
