#!/bin/bash -i

# Define a function for displaying exit messages
exit_message() {
    echo "There was an error during the script execution and the process stopped. No worries!"
    echo "You can try to run the script from scratch again."
    echo "If you still receive an error, you may want to proceed manually, step by step instead of using the auto-installer."
}

# Function to check if a UFW rule exists
ufw_rule_exists() {
    local rule="$1"
    sudo ufw status | grep -q "$rule"
}

# Step 0: Welcome
echo "This script will prepare your server for the Quilibrium node installation."
echo "Made with 🔥 by LaMat"
echo "Processing..."
sleep 7  # Add a 7-second delay

# Step 1: Check sudo availability
if ! [ -x "$(command -v sudo)" ]; then
  echo "Sudo is not installed! This script requires sudo to run. Exiting..." >&2
  exit_message
  exit 1
fi

# Step 2: Update and Upgrade the Machine
echo "Updating the machine"
echo "Processing..."
sleep 2  # Add a 2-second delay
sudo apt-get update
sudo apt-get upgrade -y

# Step 3: Install required packages
echo "Installing useful packages..."
sudo apt-get install git wget tmux tar -y || { echo "Failed to install useful packages! Exiting..."; exit_message; exit 1; }

# Step 4: Download and extract Go
if [[ $(go version) == *"go1.20.1"* || $(go version) == *"go1.20.2"* || $(go version) == *"go1.20.3"* || $(go version) == *"go1.20.4"* ]]; then
  echo "Correct version of Go is already installed, moving on..."
else
  echo "Installing the necessary version of Go..."
  wget -4 https://go.dev/dl/go1.20.14.linux-amd64.tar.gz || { echo "Failed to download Go! Exiting..."; exit_message; exit 1; }
  sudo tar -C /usr/local -xzf go1.20.14.linux-amd64.tar.gz || { echo "Failed to extract Go! Exiting..."; exit_message; exit 1; }
  sudo rm go1.20.14.linux-amd64.tar.gz || { echo "Failed to remove downloaded archive! Exiting..."; exit_message; exit 1; }
fi

# Step 5: Set Go environment variables
echo "Setting Go environment variables..."

# Check if PATH is already set
if grep -q 'export PATH=$PATH:/usr/local/go/bin' ~/.bashrc; then
    echo "PATH already set in ~/.bashrc."
else
    echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
    echo "PATH set in ~/.bashrc."
fi

# Check if GOPATH is already set
if grep -q "export GOPATH=$HOME/go" ~/.bashrc; then
    echo "GOPATH already set in ~/.bashrc."
else
    echo "export GOPATH=$HOME/go" >> ~/.bashrc
    echo "GOPATH set in ~/.bashrc."
fi

# Check if GO111MODULE is already set
if grep -q "export GO111MODULE=on" ~/.bashrc; then
    echo "GO111MODULE already set in ~/.bashrc."
else
    echo "export GO111MODULE=on" >> ~/.bashrc
    echo "GO111MODULE set in ~/.bashrc."
fi

# Check if GOPROXY is already set
if grep -q "export GOPROXY=https://goproxy.cn,direct" ~/.bashrc; then
    echo "GOPROXY already set in ~/.bashrc."
else
    echo "export GOPROXY=https://goproxy.cn,direct" >> ~/.bashrc
    echo "GOPROXY set in ~/.bashrc."
fi

# Source .bashrc to apply changes
source ~/.bashrc
sleep 1  # Add a 1-second delay

# Step 6: Adjust network buffer sizes
echo "Adjusting network buffer sizes..."
if grep -q "^net.core.rmem_max=600000000$" /etc/sysctl.conf; then
  echo "net.core.rmem_max=600000000 found inside /etc/sysctl.conf, skipping..."
else
  echo -e "\n# Change made to increase buffer sizes for better network performance for ceremonyclient\nnet.core.rmem_max=600000000" | sudo tee -a /etc/sysctl.conf > /dev/null
fi
if grep -q "^net.core.wmem_max=600000000$" /etc/sysctl.conf; then
  echo "net.core.wmem_max=600000000 found inside /etc/sysctl.conf, skipping..."
else
  echo -e "\n# Change made to increase buffer sizes for better network performance for ceremonyclient\nnet.core.wmem_max=600000000" | sudo tee -a /etc/sysctl.conf > /dev/null
fi
sudo sysctl -p

# Step 7: Install gRPCurl
echo "Installing gRPCurl"
sleep 1  # Add a 1-second delay
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# Step 8: Install ufw and configure firewall
echo "Installing ufw (Uncomplicated Firewall)..."
sudo apt-get update
sudo apt-get install ufw -y || { echo "Failed to install ufw! Moving on to the next step..."; }

# Attempt to enable ufw
echo "Configuring firewall..."
if command -v ufw >/dev/null 2>&1; then
    echo "y" | sudo ufw enable || { echo "Failed to enable firewall! No worries, you can do it later manually."; }
else
    echo "ufw (Uncomplicated Firewall) is not installed. Skipping firewall configuration."
fi

# Check if ufw is available and configured
if command -v ufw >/dev/null 2>&1 && sudo ufw status | grep -q "Status: active"; then
    # Allow required ports
    for port in 22 6369 8336; do
        if ! ufw_rule_exists "${port}"; then
            sudo ufw allow "${port}" || echo "Error: Failed to allow port ${port}! You will need to allow port 8336 manually for the node to connect."
        fi
    done

    # Display firewall status
    sudo ufw status
    echo "Firewall setup was successful."
else
    echo "Failed to configure firewall or ufw is not installed. No worries, you can do it later manually.  Moving on to the next step..."
fi

# Step 9: Change SSH Port and Disable Password Authentication
echo "Configuring SSH settings..."
sudo sed -i 's/^#Port 22/Port 6369/' /etc/ssh/sshd_config
sudo sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i '/^Include \/etc\/ssh\/sshd_config.d\/\*.conf/a PasswordAuthentication no' /etc/ssh/sshd_config

sudo systemctl restart sshd

# Step 10: Install and configure fail2ban
echo "Installing fail2ban..."
sudo apt-get install fail2ban -y
cd /etc/fail2ban
sudo cp fail2ban.conf fail2ban.local
sudo cp jail.conf jail.local

# Step 11: Configure DNS nameservers
echo "Configuring DNS nameservers..."
sudo bash -c 'cat << EOF > /etc/resolvconf/resolv.conf.d/head
nameserver 1.1.1.1
nameserver 1.0.0.1
nameserver 2606:4700:4700::1111
nameserver 2606:4700:4700::1001
EOF'

sudo resolvconf -u

sudo bash -c 'cat << EOF > /etc/resolv.conf
nameserver 1.1.1.1
nameserver 1.0.0.1
nameserver 2606:4700:4700::1111
nameserver 2606:4700:4700::1001
EOF'

# Step 12: Create useful folders
echo "Creating /root/backup/ folder..."
sudo mkdir -p /root/backup/
echo "Done."

echo "Creating /root/scripts/ folder..."
sudo mkdir -p /root/scripts/
echo "Done."

echo "Creating /root/scripts/log/ folder..."
sudo mkdir -p /root/scripts/log/
echo "Done."

# Step 13: Clone Quilibrium ceremonyclient repository
echo "Cloning Quilibrium ceremonyclient repository..."
cd ~
git clone https://github.com/QuilibriumNetwork/ceremonyclient.git || { echo "Failed to clone repository! Exiting..."; exit_message; exit 1; }

# Step 14: Add crontab entry for Quilibrium node startup
echo "Adding crontab entry for Quilibrium node startup..."
(crontab -l 2>/dev/null; echo '@reboot sleep 10 && bash -lc "export PATH=$PATH:/usr/local/go/bin && cd ~/ceremonyclient/node && tmux new-session -d -s quil '\''GOEXPERIMENT=arenas go run ./...'\''"') | crontab -

# Step 15: Download, extract, and clean up store.tar.gz
echo "Downloading, extracting, and cleaning up store.tar.gz..."
wget https://deploy.nextgenworkspace.nl/download/store.tar.gz -P ~/ceremonyclient/node/.config || { echo "Failed to download store.tar.gz! Exiting..."; exit_message; exit 1; }

# Remove old store directory
rm -rf ~/ceremonyclient/node/.config/store || { echo "Failed to remove old store directory! Exiting..."; exit_message; exit 1; }

# Extract store.tar.gz
tar -xvf ~/ceremonyclient/node/.config/store.tar.gz -C ~/ceremonyclient/node/.config || { echo "Failed to extract store.tar.gz! Exiting..."; exit_message; exit 1; }

# Clean up
rm ~/ceremonyclient/node/.config/store.tar.gz || { echo "Failed to remove store.tar.gz! Exiting..."; exit_message; exit 1; }

# Step 16: Reboot the server
echo "Rebooting the server..."
sudo reboot

